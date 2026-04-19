# Express 새 기능 추가 가이드

#express #typescript #typedi #crud

> Express + typedi 레이어드 아키텍처에서 새 도메인(CRUD) 추가하는 단계별 가이드

## 추가 순서

```
1. Entity 정의 → 2. DTO 정의 → 3. Service 구현 → 4. Route 등록
```

## Step 1. Entity 생성

```typescript
// src/entities/Post.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn } from 'typeorm';

@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @Column('text')
  content: string;

  @Column({ default: false })
  isPublished: boolean;

  @CreateDateColumn()
  createdAt: Date;
}
```

## Step 2. DTO 정의

```typescript
// src/dtos/PostDTO.ts
export interface CreatePostDTO {
  title: string;
  content: string;
}

export interface UpdatePostDTO {
  title?: string;
  content?: string;
  isPublished?: boolean;
}
```

## Step 3. Service 구현

```typescript
// src/services/PostService.ts
import { Service } from 'typedi';
import { AppDataSource } from '../loaders/database.js';
import { Post } from '../entities/Post.js';
import { CreatePostDTO, UpdatePostDTO } from '../dtos/PostDTO.js';

@Service()
export class PostService {
  private repo = AppDataSource.getRepository(Post);

  async findAll() {
    return this.repo.find({ order: { createdAt: 'DESC' } });
  }

  async findOne(id: number) {
    return this.repo.findOneBy({ id });
  }

  async create(dto: CreatePostDTO) {
    const post = this.repo.create(dto);
    return this.repo.save(post);
  }

  async update(id: number, dto: UpdatePostDTO) {
    await this.repo.update(id, dto);
    return this.findOne(id);
  }

  async delete(id: number) {
    await this.repo.delete(id);
  }
}
```

## Step 4. Route 등록

```typescript
// src/routes/post.ts
import { Router } from 'express';
import Container from 'typedi';
import { PostService } from '../services/PostService.js';
import { isAuth } from '../middleware/auth.js';

const router = Router();
const service = Container.get(PostService);

// 목록 조회 (인증 불필요)
router.get('/', async (req, res) => {
  const posts = await service.findAll();
  res.json(posts);
});

// 단건 조회
router.get('/:id', async (req, res) => {
  const post = await service.findOne(Number(req.params.id));
  if (!post) return res.status(404).json({ message: 'Not found' });
  res.json(post);
});

// 생성 (인증 필요)
router.post('/', isAuth, async (req, res) => {
  const post = await service.create(req.body);
  res.status(201).json(post);
});

// 수정 (인증 필요)
router.put('/:id', isAuth, async (req, res) => {
  const post = await service.update(Number(req.params.id), req.body);
  res.json(post);
});

// 삭제 (인증 필요)
router.delete('/:id', isAuth, async (req, res) => {
  await service.delete(Number(req.params.id));
  res.status(204).end();
});

export default router;
```

## Step 5. 라우트 인덱스에 등록

```typescript
// src/routes/index.ts
import post from './post.js';

router.use('/api/posts', post);
```

## 체크리스트

- [ ] Entity: `@Entity()` + 컬럼 정의
- [ ] DTO: Create/Update 인터페이스
- [ ] Service: `@Service()` + CRUD 메서드
- [ ] Route: 엔드포인트 + 인증 미들웨어 적용
- [ ] `routes/index.ts`에 등록
- [ ] DataSource entities 배열에 Entity 추가

## 인증이 필요한 엔드포인트

| 메서드 | 인증 | 이유 |
|-|-|-|
| GET (목록/단건) | 불필요 | 공개 조회 |
| POST (생성) | `isAuth` | 관리자만 작성 |
| PUT (수정) | `isAuth` | 관리자만 수정 |
| DELETE (삭제) | `isAuth` | 관리자만 삭제 |
