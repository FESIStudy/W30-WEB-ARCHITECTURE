## 웹서버란

웹서버는 클라이언트(주로 웹 브라우저)로부터 HTTP 요청을 받아서 해당하는 웹 페이지, 이미지, 동영상 등의 리소스를 응답으로 전송하는 역할

ec2로 인스턴스 만들어서 엔진엑스쓰는게 그 예시.

## WAS란

WAS는 동적 웹 콘텐츠를 생성하고 처리하는 미들웨어. 단순히 정적 파일을 제공하는 웹서버와 달리, 비즈니스 로직을 실행하고 데이터베이스와 연동하여 동적인 응답을 생성.

우리가 한번쯤해봤는 익스프레스로 헬로월드 찍어주기가 그거였던거임.

```js
// Express.js 서버
const express = require('express');
const app = express();

app.get('/api/users/:id', async (req, res) => {
  // DB에서 사용자 조회 (동적 처리)
  const user = await User.findById(req.params.id);
  res.json(user);
});

app.listen(3000); // Node.js가 WAS 역할
```

간단하게 생각해서 그냥 api서버가 was인거임

동적 데이터 처리
비즈니스 로직 실행
데이터베이스 연동
실시간 데이터 생성
인증/권한 처리

보통 이런구조

```
[웹서버] ← → [WAS] ← → [데이터베이스]
 Nginx      Node.js     MySQL
```

문득 든 생각

웹서버는 그러면 www.naver.com을 브라우저에 입력하면 해당하는 html과 js를 뿌려주는 역할이고
was는 뿌려줬던 js에서 api요청하면 그거 처리해주는 그런건가?

또 질문, 로드밸런싱같은거는 웹서버에서 하자나. 그러면 백엔드 api요청도 로드 밸런싱을 할 수 있나?

그러면 리버스 프록시(웹서버)를 프론트를 위해서도 백엔드를 위해서도 구성할 수 있는건가? 있겠지 근데 이것도 웹서버인가?

-> 그렇다

사용자가 www.naver.com 입력
↓
웹서버(Nginx/Apache)가 정적 파일 제공
↓
index.html, style.css, app.js 등을 브라우저로 전송

이게 웹서버의 역할인거임.

대충 웹서버의 설정값을 살펴보자잉

```yaml
# Nginx 설정
server {
    listen 80;
    server_name www.naver.com;

    location / {
        root /var/www/html;           # 정적 파일 경로
        index index.html;             # HTML 파일 제공
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg)$ { # 정적 리소스들
        root /var/www/html;
        expires 1y;                   # 캐싱
    }
}
```

was는 이런걸 한다

```js
// 브라우저에서 받은 JS가 API 요청
fetch('/api/users/123')
  .then((response) => response.json())
  .then((data) => console.log(data));

// WAS에서 처리
app.get('/api/users/:id', async (req, res) => {
  const user = await User.findById(req.params.id); // DB 조회
  res.json(user); // 동적 데이터 응답
});
```

나는 엔진엑스쓰면 무조건 웹서버라고 생각
근데 맞긴 맞음. 근데 웹서버는 html만 뿌린다라고 착각하고 있었는데
리버스 프록시로 로드밸런싱이나 리다이렉트 같은거도 가능한거임.

핵심은 HTTP요청을 처리한다는 거임.
리버스 프록시로 띄운 서버에서 어쨌는 api 요청이 http로 들어올텐데.
그 요청에 대한 리다이렉트나 로드밸런싱같은거를 처리하자나. 그니까 웹 서버인거임. 대박

클라이언트 → HTTP 요청 → Nginx → HTTP 응답 → 클라이언트
Nginx가 하는 일:

1. HTTP 요청 수신 (웹서버 기본 기능)
2. 요청 분석 및 라우팅 (어디로 보낼지 결정)
3. 백엔드 서버로 프록시 (대신 전달)
4. 응답 받아서 클라이언트로 전송

모든 과정이 HTTP 기반이므로 여전히 웹서버인거임.

```yaml
server {
    listen 80;
    server_name mysite.com;

    # 1. 웹서버 역할: 정적 파일 제공
    location /static/ {
        root /var/www;
        expires 1y;
    }

    # 2. 리버스 프록시 역할: 프론트엔드 서버로
    location / {
        proxy_pass http://frontend_servers;
    }

    # 3. 리버스 프록시 역할: 백엔드 API로
    location /api/ {
        proxy_pass http://backend_servers;
    }

    # 4. 로드 밸런서 역할: 헬스체크도 수행
    location /health {
        access_log off;
        return 200 "healthy\n";
    }
}
```

웹서버의 개념이 이런거임
옛날에는
웹서버 = 정적 파일만 제공
였다면

현재의 웹서버 (확장된 개념)

웹서버 = HTTP 요청을 처리하는 모든 소프트웨어

- 정적 파일 제공
- 리버스 프록시
- 로드 밸런싱
- SSL 터미네이션
- 캐싱
- 압축
- 보안 필터링
  라는거임.

리버스 프록시로 사용되는 Nginx
= HTTP 요청 처리 + 요청 전달
= 여전히 웹서버

백엔드 API 서버
= HTTP 요청 받아서 + 비즈니스 로직 실행
= WAS (하지만 HTTP 처리도 하므로 넓은 의미에서는 웹서버이기도 함)

그러면 was에서 db는 어떻게 연결되어가나.
이건 코드치기 귀찮으니까 지피티선생님

Nginx와 NestJS + MySQL로 예시를 바꿔서 설명해드리겠습니다!

## WAS와 DB 서버의 관계 (Nginx + NestJS + MySQL)

```
사용자 → Nginx (웹서버) → NestJS (WAS) → MySQL DB 서버
```

## 구체적인 예시

### 1. NestJS WAS에서 MySQL DB 서버로 연결

**app.module.ts (NestJS 설정)**

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserModule } from './user/user.module';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: '192.168.1.100', // MySQL DB 서버 IP
      port: 3306, // MySQL 기본 포트
      username: 'app_user',
      password: 'password123',
      database: 'myapp_db',
      entities: ['dist/**/*.entity{.ts,.js}'],
      synchronize: false, // 프로덕션에서는 false
      charset: 'utf8mb4', // MySQL 한글 지원
    }),
    UserModule,
  ],
})
export class AppModule {}
```

**user.entity.ts (MySQL 테이블 매핑)**

```typescript
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn } from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: 'varchar', length: 100 })
  name: string;

  @Column({ type: 'varchar', length: 255, unique: true })
  email: string;

  @Column({ type: 'int', default: 0 })
  age: number;

  @CreateDateColumn({ type: 'datetime' })
  created_at: Date;

  @UpdateDateColumn({ type: 'datetime' })
  updated_at: Date;
}
```

**user.controller.ts (NestJS WAS에서 실행)**

```typescript
import { Controller, Get, Post, Put, Delete, Param, Body } from '@nestjs/common';
import { UserService } from './user.service';
import { User } from './user.entity';
import { CreateUserDto, UpdateUserDto } from './dto/user.dto';

@Controller('api/users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get(':id')
  async getUser(@Param('id') id: string): Promise<User> {
    // 1. NestJS WAS에서 MySQL DB 서버로 SQL 쿼리 전송
    // SELECT * FROM users WHERE id = ?

    // 2. MySQL DB 서버가 쿼리 실행 후 결과 반환
    return await this.userService.findById(parseInt(id));
  }

  @Post()
  async createUser(@Body() createUserDto: CreateUserDto): Promise<User> {
    // INSERT INTO users (name, email, age) VALUES (?, ?, ?)
    return await this.userService.createUser(createUserDto);
  }

  @Put(':id')
  async updateUser(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto): Promise<User> {
    // UPDATE users SET name = ?, email = ?, age = ? WHERE id = ?
    return await this.userService.updateUser(parseInt(id), updateUserDto);
  }

  @Delete(':id')
  async deleteUser(@Param('id') id: string): Promise<void> {
    // DELETE FROM users WHERE id = ?
    await this.userService.deleteUser(parseInt(id));
  }
}
```

**user.service.ts (실제 MySQL 쿼리 실행)**

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';
import { CreateUserDto, UpdateUserDto } from './dto/user.dto';

@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
  ) {}

  async findById(id: number): Promise<User> {
    // TypeORM이 자동으로 MySQL SQL 생성하여 DB 서버로 전송
    // SELECT users.id, users.name, users.email, users.age, users.created_at, users.updated_at FROM users WHERE users.id = ?
    const user = await this.userRepository.findOne({
      where: { id },
    });

    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }

    return user;
  }

  async createUser(createUserDto: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(createUserDto);
    // INSERT INTO users (name, email, age, created_at, updated_at) VALUES (?, ?, ?, NOW(), NOW())
    return await this.userRepository.save(user);
  }

  async updateUser(id: number, updateUserDto: UpdateUserDto): Promise<User> {
    const user = await this.findById(id);
    Object.assign(user, updateUserDto);
    // UPDATE users SET name = ?, email = ?, age = ?, updated_at = NOW() WHERE id = ?
    return await this.userRepository.save(user);
  }

  async deleteUser(id: number): Promise<void> {
    const result = await this.userRepository.delete(id);
    // DELETE FROM users WHERE id = ?

    if (result.affected === 0) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
  }

  async findAll(): Promise<User[]> {
    // SELECT * FROM users ORDER BY created_at DESC
    return await this.userRepository.find({
      order: { created_at: 'DESC' },
    });
  }

  async findByEmail(email: string): Promise<User> {
    // SELECT * FROM users WHERE email = ?
    return await this.userRepository.findOne({
      where: { email },
    });
  }
}
```

**dto/user.dto.ts (데이터 전송 객체)**

```typescript
import { IsString, IsEmail, IsInt, IsOptional, Min, Max } from 'class-validator';

export class CreateUserDto {
  @IsString()
  name: string;

  @IsEmail()
  email: string;

  @IsInt()
  @Min(0)
  @Max(120)
  age: number;
}

export class UpdateUserDto {
  @IsOptional()
  @IsString()
  name?: string;

  @IsOptional()
  @IsEmail()
  email?: string;

  @IsOptional()
  @IsInt()
  @Min(0)
  @Max(120)
  age?: number;
}
```

### 2. Nginx 설정 (리버스 프록시)

**nginx.conf**

```nginx
upstream nestjs_backend {
    server 192.168.1.10:3000 weight=3;  # NestJS WAS 서버 1 (가중치 3)
    server 192.168.1.11:3000 weight=2;  # NestJS WAS 서버 2 (가중치 2)
    server 192.168.1.12:3000 weight=1;  # NestJS WAS 서버 3 (가중치 1)

    # 헬스체크
    server 192.168.1.13:3000 backup;    # 백업 서버

    # 로드 밸런싱 방식
    least_conn;  # 연결 수가 가장 적은 서버로
}

server {
    listen 80;
    server_name myapp.com www.myapp.com;

    # 정적 파일 제공 (Nginx가 웹서버 역할)
    location /static/ {
        root /var/www;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # API 요청을 NestJS WAS로 프록시
    location /api/ {
        proxy_pass http://nestjs_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 타임아웃 설정
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    # 프론트엔드 (React/Vue) 제공
    location / {
        root /var/www/html;
        try_files $uri $uri/ /index.html;
    }

    # 헬스체크 엔드포인트
    location /health {
        proxy_pass http://nestjs_backend/health;
        access_log off;
    }
}
```

## 실제 운영 환경 구성

### 서버 구성

```
Nginx 웹서버: 192.168.1.5:80
├── 정적 파일 제공 (/var/www/html)
├── SSL 터미네이션
├── 로드 밸런싱
└── 압축 & 캐싱

NestJS WAS 서버들:
├── 서버1: 192.168.1.10:3000 (primary)
├── 서버2: 192.168.1.11:3000 (primary)
├── 서버3: 192.168.1.12:3000 (primary)
└── 서버4: 192.168.1.13:3000 (backup)

MySQL DB 서버: 192.168.1.20:3306
├── MySQL 8.0
├── 실제 데이터 저장
├── InnoDB 스토리지 엔진
└── 쿼리 처리

Redis 캐시 서버: 192.168.1.21:6379
├── 세션 저장
├── API 응답 캐싱
└── 실시간 데이터
```

## 네트워크 통신 과정

### NestJS에서 MySQL 연결 과정

```typescript
// main.ts (NestJS 애플리케이션 시작)
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 전역 파이프 설정
  app.useGlobalPipes(new ValidationPipe());

  // CORS 설정
  app.enableCors();

  // 1. NestJS WAS가 포트 3000에서 시작
  await app.listen(3000);

  // 2. TypeORM이 자동으로 MySQL DB 서버(192.168.1.20:3306)에 연결
  console.log('🚀 NestJS WAS started on port 3000');
  console.log('🗄️  Connected to MySQL DB server at 192.168.1.20:3306');
}
bootstrap();
```

### 실제 요청 처리 흐름

```typescript
@Controller('api/users')
export class UserController {
  @Post()
  async createUser(@Body() createUserDto: CreateUserDto) {
    try {
      // 1. HTTP POST 요청이 Nginx를 통해 NestJS WAS에 도달
      console.log('📥 Received request to create user:', createUserDto);

      // 2. 유효성 검증
      // ValidationPipe가 자동으로 DTO 검증

      // 3. NestJS → MySQL DB 서버로 TCP 연결
      const user = await this.userService.createUser(createUserDto);

      // 4. SQL 쿼리가 네트워크를 통해 MySQL DB 서버로 전송
      // INSERT INTO users (name, email, age, created_at, updated_at)
      // VALUES ('홍길동', 'hong@example.com', 25, NOW(), NOW())

      // 5. MySQL DB 서버에서 쿼리 실행 후 결과를 네트워크로 전송
      // 6. NestJS WAS가 결과 수신하고 HTTP 응답으로 변환
      console.log('✅ User created successfully:', user);

      return {
        success: true,
        message: '사용자가 성공적으로 생성되었습니다.',
        data: user,
      };
    } catch (error) {
      console.error('❌ Database connection error:', error);
      return {
        success: false,
        message: 'DB 연결 실패',
        error: error.message,
      };
    }
  }
}
```

## AWS 클라우드 환경 예시

### AWS 구성

```typescript
// database.config.ts (NestJS 환경 설정)
import { TypeOrmModuleOptions } from '@nestjs/typeorm';

export const databaseConfig: TypeOrmModuleOptions = {
  type: 'mysql',
  // AWS RDS MySQL 엔드포인트
  host: 'myapp-mysql.abc123.ap-northeast-2.rds.amazonaws.com',
  port: 3306,
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: 'myapp_production',

  // SSL 연결 (AWS RDS 필수)
  ssl: {
    rejectUnauthorized: false,
  },

  // 커넥션 풀 설정
  extra: {
    connectionLimit: 20, // 최대 20개 MySQL 연결
    acquireTimeout: 30000, // 30초 연결 타임아웃
    timeout: 60000, // 60초 쿼리 타임아웃
    reconnect: true, // 자동 재연결
  },

  entities: ['dist/**/*.entity{.ts,.js}'],
  synchronize: false,
  logging: process.env.NODE_ENV === 'development',
};
```

**AWS 인프라 구성:**

```
Route 53 (DNS) → myapp.com
↓
CloudFront (CDN) → 정적 파일 캐싱
↓
Application Load Balancer (ALB)
├── Target Group: NestJS EC2 instances
│   ├── EC2-1: NestJS WAS (10.0.1.10:3000)
│   ├── EC2-2: NestJS WAS (10.0.1.11:3000)
│   └── EC2-3: NestJS WAS (10.0.1.12:3000)
├── RDS MySQL: myapp-mysql.abc123...amazonaws.com:3306
│   ├── Multi-AZ 배포 (고가용성)
│   ├── Read Replica (읽기 전용 복제본)
│   └── 자동 백업
└── ElastiCache Redis: redis.abc123...amazonaws.com:6379
```

## 커넥션 풀 관리 (NestJS + MySQL)

```typescript
// database.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: '192.168.1.20',
      port: 3306,
      username: 'app_user',
      password: 'password123',
      database: 'myapp_db',

      // MySQL 커넥션 풀 상세 설정
      extra: {
        connectionLimit: 20, // 최대 20개 연결
        queueLimit: 0, // 무제한 대기열
        acquireTimeout: 30000, // 연결 획득 타임아웃 30초
        timeout: 60000, // 쿼리 타임아웃 60초
        reconnect: true, // 연결 끊어지면 자동 재연결
        charset: 'utf8mb4', // 한글, 이모지 지원
      },

      entities: ['dist/**/*.entity{.ts,.js}'],
      synchronize: false,
    }),
  ],
})
export class DatabaseModule {}
```

## 전체 아키텍처 (Nginx + NestJS + MySQL)

```
사용자 (브라우저)
  ↓ HTTP POST /api/users

Nginx 웹서버 (192.168.1.5:80)
  ├── SSL 인증서 처리 (HTTPS)
  ├── 정적 파일 제공 (/static/)
  ├── 압축 (gzip)
  └── 로드 밸런싱 (least_conn)
  ↓ proxy_pass http://nestjs_backend

NestJS WAS (192.168.1.10:3000)
  ├── @Controller에서 HTTP 요청 수신
  ├── ValidationPipe로 데이터 검증
  ├── @Service에서 비즈니스 로직 처리
  └── TypeORM으로 MySQL 쿼리 실행
  ↓ TCP/IP: INSERT INTO users (name, email, age) VALUES (?, ?, ?)

MySQL DB 서버 (192.168.1.20:3306)
  ├── InnoDB 엔진으로 트랜잭션 처리
  ├── SQL 쿼리 실행 및 데이터 저장
  ├── 인덱스 기반 빠른 조회
  └── 결과 반환
  ↓ TCP/IP: { insertId: 123, affectedRows: 1 }

NestJS WAS
  ├── MySQL 결과 수신
  ├── Entity 객체로 변환
  ├── DTO로 응답 데이터 구성
  └── HTTP JSON 응답 생성
  ↓ HTTP 201 Created + JSON

Nginx 웹서버
  ├── 응답 수신 및 압축
  ├── 로깅 (access.log)
  └── 클라이언트로 전달
  ↓ HTTP 응답

사용자 (브라우저)
  └── JSON 응답 수신 및 UI 업데이트
```

**핵심:**

- **Nginx**: 웹서버 (HTTP 처리, 정적 파일, 프록시, SSL)
- **NestJS**: WAS (HTTP API, 비즈니스 로직, 유효성 검증)
- **MySQL**: DB 서버 (데이터 저장/조회, 트랜잭션)
- 모두 **네트워크(TCP/IP)로 분리된 독립적인 서버/프로세스**
