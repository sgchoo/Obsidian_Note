[[Nest.js]]

## 문제 상황

다음 Prisma 쿼리에서 `orderBy`의 `case` 구문이 작동하지 않는 문제가 발생했습니다:

```typescript
const adminList = await this.prisma.admin.findMany({
  where: {},
  select: {
    id: true,
    email: true,
    name: true,
    nickname: true,
    phone: true,
    platform: true,
    role: true,
    status: true,
    createdAt: true,
    updatedAt: true,
    deletedAt: true,
  },
  orderBy: {
    case: {
      when: {
        status: AdminStatus.INACTIVE,
      },
      then: 0,
      else: 1,
    },
    createdAt: "asc",
  },
  skip: 0,
  take: 10,
});
```

## 원인

Prisma는 현재 `orderBy`에서 직접적인 `case` 구문을 지원하지 않습니다. 이러한 복잡한 정렬 로직은 Prisma의 기본 쿼리 문법으로는 표현하기 어렵습니다.

## 해결 방법

### 1. 기본 정렬 방식 사용

`status`를 기준으로 먼저 정렬하고, 그 다음으로 `createdAt`으로 정렬:

```typescript
const adminList = await this.prisma.admin.findMany({
  where: {},
  select: {
    id: true,
    email: true,
    name: true,
    nickname: true,
    phone: true,
    platform: true,
    role: true,
    status: true,
    createdAt: true,
    updatedAt: true,
    deletedAt: true,
  },
  orderBy: [
    { status: "asc" }, // INACTIVE가 더 낮은 숫자라면 먼저 표시됨
    { createdAt: "asc" },
  ],
  skip: 0,
  take: 10,
});
```

### 2. JavaScript로 추가 정렬

결과를 가져온 후 JavaScript 코드로 사용자 정의 정렬 로직 적용:

```typescript
let adminList = await this.prisma.admin.findMany({
  where: {},
  select: {
    id: true,
    email: true,
    name: true,
    nickname: true,
    phone: true,
    platform: true,
    role: true,
    status: true,
    createdAt: true,
    updatedAt: true,
    deletedAt: true,
  },
  orderBy: {
    createdAt: "asc",
  },
  skip: 0,
  take: 10,
});

// 비활성 계정을 먼저 정렬
adminList = adminList.sort((a, b) => {
  if (a.status === AdminStatus.INACTIVE && b.status !== AdminStatus.INACTIVE) return -1;
  if (a.status !== AdminStatus.INACTIVE && b.status === AdminStatus.INACTIVE) return 1;
  
  // status가 같은 경우 createdAt으로 정렬
  return new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime();
});
```

### 3. Raw SQL 쿼리 사용

SQL의 `CASE` 구문을 활용하여 직접 쿼리 작성:

```typescript
const adminList = await this.prisma.$queryRaw`
  SELECT id, email, name, nickname, phone, platform, role, status, "createdAt", "updatedAt", "deletedAt"
  FROM "Admin"
  ORDER BY 
    CASE WHEN status = ${AdminStatus.INACTIVE} THEN 0 ELSE 1 END,
    "createdAt" ASC
  LIMIT 10
`;
```

## 참고사항

- Raw SQL 쿼리는 데이터베이스에 따라 문법이 다를 수 있으며, 타입 안전성이 떨어집니다.
- 첫 번째 방법(기본 정렬 방식 사용)이 가장 간단하고 Prisma의 기능을 최대한 활용합니다.
- Prisma는 지속적으로 업데이트되고 있으므로, 향후 버전에서는 더 복잡한 정렬 로직이 지원될 수 있습니다.