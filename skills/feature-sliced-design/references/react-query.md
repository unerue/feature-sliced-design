# FSD with React Query

Guide for using React Query with Feature-Sliced Design.  
Source: [Usage with React Query](https://feature-sliced.design/docs/guides/tech/with-react-query)

---

## The problem of "where to put the keys"

### Solution — break down by entities

If the project already has a division into entities, and each request corresponds to a single entity, the purest division will be by entity.

```
└── src/
    ├── app/
    |   ...
    ├── pages/
    |   ...
    ├── entities/
    |     ├── {entity}/
    |    ...     └── api/
    |                 ├── `{entity}.query`      # Query-factory (keys + functions)
    |                 ├── `get-{entity}`        # Entity getter
    |                 ├── `create-{entity}`     # Entity creation
    |                 ├── `update-{entity}`     # Entity update
    |                 ├── `delete-{entity}`     # Entity delete
    |                ...
    |
    ├── features/
    |   ...
    ├── widgets/
    |   ...
    └── shared/
        ...
```

If there are connections between entities (e.g., Country has a list of City entities), use the public API for cross-imports or consider the alternative below.

### Alternative solution — keep it in shared

In cases where entity separation is not appropriate:

```
└── src/
   ...
    └── shared/
          ├── api/
         ...   ├── `queries`
               |      ├── `document.ts`
               |      ├── `background-jobs.ts`
               |     ...
               └──  index.ts
```

`@/shared/api/index.ts`:

```typescript
export { documentQueries } from "./queries/document";
```

---

## The problem of "Where to insert mutations?"

It is not recommended to mix mutations with queries. There are two options:

### 1. Define a custom hook in the `api` segment near the place of use

`@/features/update-post/api/use-update-title.ts`:

```typescript
export const useUpdateTitle = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, newTitle }) =>
      apiClient
        .patch(`/posts/${id}`, { title: newTitle })
        .then((data) => console.log(data)),

    onSuccess: (newPost) => {
      queryClient.setQueryData(postsQueries.ids(id), newPost);
    },
  });
};
```

### 2. Define a mutation function somewhere else (Shared or Entities) and use `useMutation` directly in the component

```typescript
const { mutateAsync, isPending } = useMutation({
  mutationFn: postApi.createPost,
});
```

`@/pages/post-create/ui/post-create-page.tsx`:

```typescript
export const CreatePost = () => {
  const { classes } = useStyles();
  const [title, setTitle] = useState("");

  const { mutate, isPending } = useMutation({
    mutationFn: postApi.createPost,
  });

  const handleChange = (e: ChangeEvent<HTMLInputElement>) =>
    setTitle(e.target.value);
  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    mutate({ title, userId: DEFAULT_USER_ID });
  };

  return (
    <form className={classes.create_form} onSubmit={handleSubmit}>
      <TextField onChange={handleChange} value={title} />
      <LoadingButton type="submit" variant="contained" loading={isPending}>
        Create
      </LoadingButton>
    </form>
  );
};
```

---

## Organization of requests

### Query factory

A query factory is an object where the key values are functions that return a list of query keys.

```typescript
const keyFactory = {
  all: () => ["entity"],
  lists: () => [...postQueries.all(), "list"],
};
```

> **info:** `queryOptions` is a built-in utility in react-query@v5 (optional)

```typescript
queryOptions({
  queryKey,
  ...options,
});
```

For greater type safety and compatibility with future versions of react-query, use the built-in `queryOptions` function from `@tanstack/react-query`.

### 1. Creating a Query Factory

`@/entities/post/api/post.queries.ts`:

```typescript
import { keepPreviousData, queryOptions } from "@tanstack/react-query";
import { getPosts } from "./get-posts";
import { getDetailPost } from "./get-detail-post";
import { PostDetailQuery } from "./query/post.query";

export const postQueries = {
  all: () => ["posts"],

  lists: () => [...postQueries.all(), "list"],
  list: (page: number, limit: number) =>
    queryOptions({
      queryKey: [...postQueries.lists(), page, limit],
      queryFn: () => getPosts(page, limit),
      placeholderData: keepPreviousData,
    }),

  details: () => [...postQueries.all(), "detail"],
  detail: (query?: PostDetailQuery) =>
    queryOptions({
      queryKey: [...postQueries.details(), query?.id],
      queryFn: () => getDetailPost({ id: query?.id }),
      staleTime: 5000,
    }),
};
```

### 2. Using Query Factory in application code

```typescript
import { useParams } from "react-router-dom";
import { postApi } from "@/entities/post";
import { useQuery } from "@tanstack/react-query";

type Params = {
  postId: string;
};

export const PostPage = () => {
  const { postId } = useParams<Params>();
  const id = parseInt(postId || "");
  const {
    data: post,
    error,
    isLoading,
    isError,
  } = useQuery(postApi.postQueries.detail({ id }));

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (isError || !post) {
    return <>{error?.message}</>;
  }

  return (
    <div>
      <p>Post id: {post.id}</p>
      <div>
        <h1>{post.title}</h1>
        <div><p>{post.body}</p></div>
      </div>
      <div>Owner: {post.userId}</div>
    </div>
  );
};
```

### Benefits of using a Query Factory

- **Request structuring:** Organize all API requests in one place for better readability and maintainability
- **Convenient access to queries and keys:** Easy access to different query types and their keys
- **Query refetching ability:** Easy refetching without changing query keys across the application

---

## Pagination

### 1. Creating a function `getPosts`

`@/pages/post-feed/api/get-posts.ts`:

```typescript
import { apiClient } from "@/shared/api/base";
import { PostWithPaginationDto } from "./dto/post-with-pagination.dto";
import { PostQuery } from "./query/post.query";
import { mapPost } from "./mapper/map-post";
import { PostWithPagination } from "../model/post-with-pagination";

const calculatePostPage = (totalCount: number, limit: number) =>
  Math.floor(totalCount / limit);

export const getPosts = async (
  page: number,
  limit: number,
): Promise<PostWithPagination> => {
  const skip = page * limit;
  const query: PostQuery = { skip, limit };
  const result = await apiClient.get<PostWithPaginationDto>("/posts", query);

  return {
    posts: result.posts.map((post) => mapPost(post)),
    limit: result.limit,
    skip: result.skip,
    total: result.total,
    totalPages: calculatePostPage(result.total, limit),
  };
};
```

### 2. Query factory for pagination

```typescript
import { keepPreviousData, queryOptions } from "@tanstack/react-query";
import { getPosts } from "./get-posts";

export const postQueries = {
  all: () => ["posts"],
  lists: () => [...postQueries.all(), "list"],
  list: (page: number, limit: number) =>
    queryOptions({
      queryKey: [...postQueries.lists(), page, limit],
      queryFn: () => getPosts(page, limit),
      placeholderData: keepPreviousData,
    }),
};
```

### 3. Use in application code

`@/pages/home/ui/index.tsx`:

```typescript
export const HomePage = () => {
  const itemsOnScreen = DEFAULT_ITEMS_ON_SCREEN;
  const [page, setPage] = usePageParam(DEFAULT_PAGE);
  const { data, isFetching, isLoading } = useQuery(
    postApi.postQueries.list(page, itemsOnScreen),
  );
  return (
    <>
      <Pagination
        onChange={(_, page) => setPage(page)}
        page={page}
        count={data?.totalPages}
        variant="outlined"
        color="primary"
      />
      <Posts posts={data?.posts} />
    </>
  );
};
```

---

## QueryProvider for managing queries

### 1. Creating a QueryProvider

`@/app/providers/query-provider.tsx`:

```typescript
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { ReactNode } from "react";

type Props = {
  children: ReactNode;
  client: QueryClient;
};

export const QueryProvider = ({ client, children }: Props) => {
  return (
    <QueryClientProvider client={client}>
      {children}
      <ReactQueryDevtools />
    </QueryClientProvider>
  );
};
```

### 2. Creating a QueryClient

`@/shared/api/query-client.ts`:

```typescript
import { QueryClient } from "@tanstack/react-query";

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      gcTime: 5 * 60 * 1000,
    },
  },
});
```

---

## Code generation

There are tools that can generate API code for you (e.g., from Swagger), but they are less flexible than the manual approach. If your Swagger file is well-structured, consider generating all the code in the `@/shared/api` directory.

---

## Additional advice for organizing RQ

### API Client

Using a custom API client class in the shared layer allows you to standardize logging, headers, and data exchange format (such as JSON or XML) from one place.

`@/shared/api/api-client.ts`:

```typescript
import { API_URL } from "@/shared/config";

export class ApiClient {
  private baseUrl: string;

  constructor(url: string) {
    this.baseUrl = url;
  }

  async handleResponse<TResult>(response: Response): Promise<TResult> {
    if (!response.ok) {
      throw new Error(`HTTP error! Status: ${response.status}`);
    }
    try {
      return await response.json();
    } catch (error) {
      throw new Error("Error parsing JSON response");
    }
  }

  public async get<TResult = unknown>(
    endpoint: string,
    queryParams?: Record<string, string | number>,
  ): Promise<TResult> {
    const url = new URL(endpoint, this.baseUrl);
    if (queryParams) {
      Object.entries(queryParams).forEach(([key, value]) => {
        url.searchParams.append(key, value.toString());
      });
    }
    const response = await fetch(url.toString(), {
      method: "GET",
      headers: { "Content-Type": "application/json" },
    });
    return this.handleResponse<TResult>(response);
  }

  public async post<TResult = unknown, TData = Record<string, unknown>>(
    endpoint: string,
    body: TData,
  ): Promise<TResult> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
    return this.handleResponse<TResult>(response);
  }
}

export const apiClient = new ApiClient(API_URL);
```

---

## See also

- [About the query factory](https://tkdodo.eu/blog/the-query-options-api)
- [(CodeSandbox) Sample Project](https://codesandbox.io/p/github/ruslan4432013/fsd-react-query-example/main)
- [(GitHub) Sample Project](https://github.com/ruslan4432013/fsd-react-query-example)
