[[DBMS]]

# PostgreSQL High Availability 구성 가이드

## 📖 개요

단일 서버 인스턴스에서 PostgreSQL의 고가용성(HA)과 읽기/쓰기 분리를 구현하는 완전한 가이드입니다.

## 🏗️ 아키텍처 구성

### 전체 구조

```
Application (NestJS/Prisma)
    ↓
pgbouncer (Connection Pooling)
    ↓  
pgpool-II (Query Routing & Load Balancing)
    ↓
├── Write DB (Primary) - 쓰기 전용
├── Read DB (Secondary) - 읽기 전용
└── Standby DB - 장애 대응 대기
    ↑
repmgr (Monitoring & Auto Failover)
```

### 포트 구성

- **pgbouncer**: 6432 (애플리케이션 연결점)
- **pgpool-II**: 5430 (쿼리 라우팅)
- **Write DB**: 5432 (Primary)
- **Read DB**: 5433 (Secondary)
- **Standby DB**: 5434 (대기)

## 🛠️ 구성 요소 역할

### 1. repmgr (Replication Manager)

- **역할**: DB 복제 관리 및 자동 failover
- **기능**:
    - PostgreSQL 클러스터 생명주기 관리
    - 노드 상태 모니터링
    - 장애 시 자동 승격

### 2. pgpool-II (Connection Pool + Load Balancer)

- **역할**: 쿼리 라우팅 및 로드밸런싱
- **기능**:
    - 읽기/쓰기 쿼리 자동 분산
    - DB 헬스체크
    - 연결 관리

### 3. pgbouncer (Connection Pooler)

- **역할**: 연결 풀링 및 성능 최적화
- **기능**:
    - 연결 수 제한 및 재사용
    - 트랜잭션/세션 단위 풀링
    - DB 부하 감소

## 🐳 Docker Compose 구성

```yaml
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: dev
    container_name: aratax-server-dev
    volumes:
      - .:/app
    env_file:
      - .env.dev
    depends_on:
      - pgbouncer
    networks:
      - aratax-network
    ports:
      - "8080:8080"

  # Primary DB (Write)
  primary:
    image: bitnami/postgresql:15
    container_name: postgres-primary
    ports:
      - 5432:5432
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRESQL_REPLICATION_MODE=master
      - POSTGRESQL_REPLICATION_USER=${POSTGRESQL_REPLICATION_USER}
      - POSTGRESQL_REPLICATION_PASSWORD=${POSTGRESQL_REPLICATION_PASSWORD}
    volumes:
      - primary_data:/bitnami/postgresql
    networks:
      - aratax-network

  # Secondary DB (Read)
  secondary:
    image: bitnami/postgresql:15
    container_name: postgres-secondary
    ports:
      - 5433:5432
    depends_on:
      - primary
    environment:
      - POSTGRESQL_REPLICATION_MODE=slave
      - POSTGRESQL_MASTER_HOST=primary
      - POSTGRESQL_MASTER_PORT_NUMBER=5432
      - POSTGRESQL_REPLICATION_USER=${POSTGRESQL_REPLICATION_USER}
      - POSTGRESQL_REPLICATION_PASSWORD=${POSTGRESQL_REPLICATION_PASSWORD}
      - POSTGRESQL_USERNAME=${POSTGRES_USER}
      - POSTGRESQL_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - secondary_data:/bitnami/postgresql
    networks:
      - aratax-network

  # Standby DB (Failover)
  standby:
    image: bitnami/postgresql:15
    container_name: postgres-standby
    ports:
      - 5434:5432
    depends_on:
      - primary
    environment:
      - POSTGRESQL_REPLICATION_MODE=slave
      - POSTGRESQL_MASTER_HOST=primary
      - POSTGRESQL_MASTER_PORT_NUMBER=5432
      - POSTGRESQL_REPLICATION_USER=${POSTGRESQL_REPLICATION_USER}
      - POSTGRESQL_REPLICATION_PASSWORD=${POSTGRESQL_REPLICATION_PASSWORD}
      - POSTGRESQL_USERNAME=${POSTGRES_USER}
      - POSTGRESQL_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - standby_data:/bitnami/postgresql
    networks:
      - aratax-network

  # pgpool-II (Query Routing)
  pgpool:
    image: bitnami/pgpool:4
    container_name: pgpool
    ports:
      - 5430:5432
    depends_on:
      - primary
      - secondary
      - standby
    environment:
      - PGPOOL_BACKEND_NODES=0:primary:5432:1:primary:primary,1:secondary:5432:1:replica:replica,2:standby:5432:1:replica:standby
      - PGPOOL_SR_CHECK_USER=${POSTGRESQL_REPLICATION_USER}
      - PGPOOL_SR_CHECK_PASSWORD=${POSTGRESQL_REPLICATION_PASSWORD}
      - PGPOOL_POSTGRES_USERNAME=${POSTGRES_USER}
      - PGPOOL_POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - PGPOOL_ADMIN_USERNAME=admin
      - PGPOOL_ADMIN_PASSWORD=admin123
      - PGPOOL_ENABLE_LOAD_BALANCING=yes
      - PGPOOL_ENABLE_STATEMENT_LOAD_BALANCING=yes
    networks:
      - aratax-network

  # pgbouncer (Connection Pooling)
  pgbouncer:
    image: bitnami/pgbouncer:1
    container_name: pgbouncer
    ports:
      - 6432:6432
    depends_on:
      - pgpool
    environment:
      - PGBOUNCER_DATABASE_HOST=pgpool
      - PGBOUNCER_DATABASE_PORT=5432
      - PGBOUNCER_DATABASE_USER=${POSTGRES_USER}
      - PGBOUNCER_DATABASE_PASSWORD=${POSTGRES_PASSWORD}
      - PGBOUNCER_DATABASE_NAME=${POSTGRES_DB}
      - PGBOUNCER_PORT=6432
      - PGBOUNCER_POOL_MODE=transaction
    networks:
      - aratax-network

networks:
  aratax-network:
    name: aratax-network
    external: true

volumes:
  primary_data:
    name: primary_data
  secondary_data:
    name: secondary_data
  standby_data:
    name: standby_data
```

## ⚙️ 환경 변수 설정

### .env 파일

```env
# PostgreSQL 기본 설정
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mypassword
POSTGRES_DB=mydb

# Replication 설정
POSTGRESQL_REPLICATION_USER=replicator
POSTGRESQL_REPLICATION_PASSWORD=replicatorpass
POSTGRESQL_MASTER_PORT_NUMBER=5432
```

### .env.dev 파일 (애플리케이션용)

```env
# pgbouncer를 통한 연결
DATABASE_URL="postgresql://myuser:mypassword@pgbouncer:6432/mydb"
```

## 💻 애플리케이션 코드 (NestJS + Prisma)

### Prisma Schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id    Int    @id @default(autoincrement())
  name  String
  email String @unique
  posts Post[]
}

model Post {
  id       Int    @id @default(autoincrement())
  title    String
  content  String
  authorId Int
  author   User   @relation(fields: [authorId], references: [id])
}
```

### Service 코드

```typescript
@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  // 자동으로 Read DB로 전달
  async findAll() {
    return await this.prisma.user.findMany({
      include: { posts: true }
    });
  }

  // 자동으로 Write DB로 전달
  async create(data: { name: string; email: string }) {
    return await this.prisma.user.create({ data });
  }

  // 트랜잭션 - 모든 쿼리가 Write DB로 전달
  async transferData() {
    return await this.prisma.$transaction(async (prisma) => {
      // 트랜잭션 내 모든 쿼리는 Write DB 사용
      const user = await prisma.user.findUnique({ where: { id: 1 } });
      await prisma.user.update({ where: { id: 1 }, data: { name: "Updated" } });
      return user;
    });
  }
}
```

## 🔄 쿼리 라우팅 동작

### 자동 라우팅 규칙

|쿼리 타입|대상 DB|예시|
|---|---|---|
|**SELECT**|Read DB|`SELECT * FROM users`|
|**INSERT**|Write DB|`INSERT INTO users VALUES (...)`|
|**UPDATE**|Write DB|`UPDATE users SET name = ...`|
|**DELETE**|Write DB|`DELETE FROM users WHERE ...`|
|**트랜잭션**|Write DB|`BEGIN; ... COMMIT;`|

### 장애 시나리오

1. **Write DB 장애**: repmgr이 Standby DB를 새로운 Primary로 승격
2. **Read DB 장애**: pgpool-II가 읽기 트래픽을 Standby DB로 라우팅
3. **자동 복구**: 장애 노드 복구 시 자동으로 클러스터에 재참여

## 💾 저장공간 고려사항

### 물리적 복제 저장공간

```
Write DB:   100GB
Read DB:    100GB (전체 복제)
Standby DB: 100GB (전체 복제)
Total:      ~300GB
```

### 절약 방법

- **Logical Replication**: 필요한 테이블만 복제
- **압축 설정**: WAL 압축 활성화
- **부분 복제**: 중요한 데이터만 복제

## 🚀 실행 방법

### 1. 네트워크 생성

```bash
docker network create aratax-network
```

### 2. 서비스 시작

```bash
docker-compose up -d
```

### 3. 복제 상태 확인

```sql
-- Primary DB에서 실행
SELECT client_addr, state, sync_state FROM pg_stat_replication;

-- Secondary/Standby DB에서 실행  
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

## ⚠️ 주의사항

### Read After Write 문제

```typescript
// 문제 상황
const user = await prisma.user.create({ data: userData }); // Write DB
const found = await prisma.user.findUnique({ where: { id: user.id } }); // Read DB (복제 지연)

// 해결 방법: 트랜잭션 사용
const result = await prisma.$transaction(async (prisma) => {
  const user = await prisma.user.create({ data: userData });
  const found = await prisma.user.findUnique({ where: { id: user.id } });
  return { user, found };
});
```

### 모니터링 포인트

- 복제 지연 시간
- 연결 풀 사용률
- 각 DB별 부하 분산 상태
- Failover 발생 빈도

## 🎯 장점

✅ **완전한 자동 failover**  
✅ **투명한 읽기/쓰기 분리**  
✅ **연결 풀링으로 성능 향상**  
✅ **애플리케이션 코드 변경 최소화**  
✅ **Docker 환경에서 쉬운 관리**

## 📚 참고 자료

- [PostgreSQL Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [Bitnami PostgreSQL HA](https://github.com/bitnami/charts/tree/main/bitnami/postgresql-ha)
- [pgpool-II Documentation](https://www.pgpool.net/docs/latest/en/html/)
- [pgbouncer Documentation](https://www.pgbouncer.org/)
- [repmgr Documentation](https://repmgr.org/docs/)