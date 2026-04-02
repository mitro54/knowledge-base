# Code Organization

## Title & Summary

Code organization encompasses strategies for structuring files, directories, modules, and packages within a codebase. Effective organization improves discoverability, maintainability, and scalability while enabling teams to navigate and understand large codebases efficiently.

## Problem Statement

Poor code organization leads to:
- **Lost files**: Difficulty finding where code lives
- **Unclear dependencies**: Uncertain how modules relate
- **Merge conflicts**: Related changes scattered across files
- **Onboarding friction**: New developers can't find anything
- **Refactoring nightmares**: Unclear impact of changes
- **Duplicate code**: Can't find existing functionality

## Solution

### Organizational Principles

**1. Colocation Principle**
Place related code together. Things that change together should be together.

```
// ❌ Poor colocation - Related code scattered
/src/
  users.js          # User logic
  orders.js         # Order logic  
  user-validation.js # User validation (should be with users)
  order-utils.js    # Order utilities (should be with orders)

// ✅ Good colocation - Related code together
/src/
  user/
    user.js
    user-validation.js
    user-utils.js
  order/
    order.js
    order-utils.js
```

**2. Single Responsibility at File Level**
Each file should have one clear purpose.

```
// ❌ Multiple responsibilities in one file
// user.js - Does everything
class User { }
function validateUser() { }
function formatUser() { }
function saveUserToDb() { }
function sendUserEmail() { }

// ✅ Single responsibility per file
// user.model.js
class User { }

// user-validator.js
function validateUser() { }

// user-formatter.js  
function formatUser() { }
```

### Common Organization Patterns

**Feature-Based Organization**

Organize by feature/domain. Best for medium to large applications.

```
src/
├── auth/
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── auth.repository.ts
│   ├── auth.types.ts
│   └── auth.constants.ts
├── users/
│   ├── user.controller.ts
│   ├── user.service.ts
│   ├── user.repository.ts
│   ├── user.types.ts
│   └── user.validators.ts
├── products/
│   ├── product.controller.ts
│   ├── product.service.ts
│   ├── product.repository.ts
│   └── product.types.ts
├── orders/
│   ├── order.controller.ts
│   ├── order.service.ts
│   ├── order.repository.ts
│   └── order.types.ts
├── shared/
│   ├── utils/
│   ├── middleware/
│   └── types/
└── app.ts
```

**Layer-Based Organization**

Organize by technical layer. Good for smaller applications or teams organized by layer.

```
src/
├── controllers/
│   ├── auth.controller.ts
│   ├── user.controller.ts
│   ├── product.controller.ts
│   └── order.controller.ts
├── services/
│   ├── auth.service.ts
│   ├── user.service.ts
│   ├── product.service.ts
│   └── order.service.ts
├── repositories/
│   ├── auth.repository.ts
│   ├── user.repository.ts
│   ├── product.repository.ts
│   └── order.repository.ts
├── models/
│   ├── user.model.ts
│   ├── product.model.ts
│   └── order.model.ts
├── middleware/
│   ├── auth.middleware.ts
│   └── validation.middleware.ts
└── utils/
    ├── validation.ts
    └── helpers.ts
```

**Domain-Driven Design Organization**

Organize by bounded contexts with clear domain boundaries.

```
src/
├── domains/
│   ├── identity/
│   │   ├── entities/
│   │   ├── value-objects/
│   │   ├── repositories/
│   │   ├── services/
│   │   └── events/
│   ├── catalog/
│   │   ├── entities/
│   │   ├── value-objects/
│   │   ├── repositories/
│   │   ├── services/
│   │   └── events/
│   └── ordering/
│       ├── entities/
│       ├── value-objects/
│       ├── repositories/
│       ├── services/
│       └── events/
├── application/
│   ├── commands/
│   ├── queries/
│   └── dtos/
├── infrastructure/
│   ├── persistence/
│   ├── messaging/
│   └── external/
└── presentation/
    ├── controllers/
    └── serializers/
```

### File Organization Patterns

**Barrel Files (Index Pattern)**

Centralize exports for cleaner imports.

```
// users/index.ts - Barrel file
export { UserService } from './user.service';
export { UserRepository } from './user.repository';
export { User } from './user.model';
export type { CreateUserDto, UpdateUserDto } from './user.types';

// Clean import elsewhere
import { UserService, UserRepository, User } from './users';
// Instead of:
// import { UserService } from './users/user.service';
// import { UserRepository } from './users/user.repository';
// import { User } from './users/user.model';
```

**Module Pattern**

Self-contained modules with clear boundaries.

```typescript
// users/user.module.ts
import { UserService } from './user.service';
import { UserRepository } from './user.repository';

export function createUserModule() {
    const userRepository = new UserRepository();
    const userService = new UserService(userRepository);
    
    return {
        getUser: userService.getUser.bind(userService),
        createUser: userService.createUser.bind(userService),
        updateUser: userService.updateUser.bind(userService),
        deleteUser: userService.deleteUser.bind(userService),
    };
}

// Usage
const userModule = createUserModule();
const user = await userModule.getUser('123');
```

### Directory Structure by Project Type

**Node.js Backend API**

```
my-api/
├── src/
│   ├── config/
│   │   ├── database.config.ts
│   │   ├── redis.config.ts
│   │   └── app.config.ts
│   ├── database/
│   │   ├── connection.ts
│   │   ├── migrations/
│   │   └── seeds/
│   ├── modules/
│   │   ├── auth/
│   │   ├── users/
│   │   ├── products/
│   │   └── orders/
│   ├── shared/
│   │   ├── decorators/
│   │   ├── filters/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   └── pipes/
│   ├── utils/
│   │   ├── logger.ts
│   │   ├── validators.ts
│   │   └── helpers.ts
│   ├── main.ts
│   └── app.module.ts
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── scripts/
├── .env
├── .env.example
├── package.json
└── tsconfig.json
```

**React Frontend Application**

```
my-react-app/
├── src/
│   ├── components/
│   │   ├── common/
│   │   │   ├── Button/
│   │   │   ├── Input/
│   │   │   └── Modal/
│   │   └── layout/
│   │       ├── Header/
│   │       ├── Sidebar/
│   │       └── Footer/
│   ├── features/
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── slices/
│   │   │   └── services/
│   │   └── dashboard/
│   ├── hooks/
│   ├── services/
│   ├── store/
│   ├── utils/
│   ├── constants/
│   ├── types/
│   ├── styles/
│   ├── routes/
│   ├── pages/
│   ├── App.tsx
│   └── main.tsx
├── public/
├── tests/
├── package.json
└── tsconfig.json
```

**Full-Stack Monorepo**

```
my-monorepo/
├── packages/
│   ├── api/
│   │   ├── src/
│   │   ├── tests/
│   │   └── package.json
│   ├── web/
│   │   ├── src/
│   │   ├── tests/
│   │   └── package.json
│   ├── shared/
│   │   ├── src/
│   │   ├── types/
│   │   ├── utils/
│   │   └── package.json
│   └── database/
│       ├── src/
│       ├── migrations/
│       └── package.json
├── tools/
├── scripts/
├── turbo.json
├── package.json
└── pnpm-workspace.yaml
```

## When to Use

| Pattern | Best For |
|--------|---------|
| **Feature-Based** | Medium-large apps, feature teams |
| **Layer-Based** | Small apps, layer-specialized teams |
| **DDD** | Complex domains, enterprise apps |
| **Barrel Files** | Any project wanting clean imports |
| **Module Pattern** | Dependency injection, testing |

## Tradeoffs

| Pattern | Pros | Cons |
|--------|-----|-----|
| **Feature-Based** | Easy to find related code, good for features | Duplication across features |
| **Layer-Based** | Clear separation, easy to swap layers | Related code scattered |
| **DDD** | Models domain well, clear boundaries | Complex, steep learning curve |
| **Barrel Files** | Clean imports, encapsulation | Can hide circular deps |

## Implementation Example

### Feature-Based E-Commerce Backend

```typescript
// src/modules/users/users.module.ts
import { Module } from '@nestjs/module';
import { UsersController } from './users.controller';
import { UserService } from './users.service';
import { UserRepository } from './users.repository';

@Module({
    controllers: [UsersController],
    providers: [UserService, UserRepository],
    exports: [UserService],
})
export class UsersModule {}

// src/modules/users/users.controller.ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { UserService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';

@Controller('users')
export class UsersController {
    constructor(private userService: UserService) {}
    
    @Post()
    create(@Body() dto: CreateUserDto) {
        return this.userService.create(dto);
    }
    
    @Get(':id')
    findById(@Param('id') id: string) {
        return this.userService.findById(id);
    }
}

// src/modules/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { UserRepository } from './users.repository';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UserService {
    constructor(private repository: UserRepository) {}
    
    async create(dto: CreateUserDto) {
        // Business logic
        return this.repository.save(dto);
    }
    
    async findById(id: string) {
        return this.repository.findOne(id);
    }
}

// src/modules/users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { User } from './entities/user.entity';

@Injectable()
export class UserRepository {
    constructor(
        @InjectRepository(User)
        private repo: Repository<User>
    ) {}
    
    save(data) { return this.repo.save(data); }
    findOne(id) { return this.repo.findOne(id); }
}

// src/modules/users/dto/create-user.dto.ts
import { IsEmail, IsString } from 'class-validator';

export class CreateUserDto {
    @IsString()
    name: string;
    
    @IsEmail()
    email: string;
}

// src/modules/users/entities/user.entity.ts
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity('users')
export class User {
    @PrimaryGeneratedColumn('uuid')
    id: string;
    
    @Column()
    name: string;
    
    @Column({ unique: true })
    email: string;
}

// src/app.module.ts - Root module
import { Module } from '@nestjs/module';
import { UsersModule } from './modules/users/users.module';
import { ProductsModule } from './modules/products/products.module';
import { OrdersModule } from './modules/orders/orders.module';

@Module({
    imports: [UsersModule, ProductsModule, OrdersModule],
})
export class AppModule {}
```

## Anti-Pattern

### The "Utils Hell" Anti-Pattern

```
// ❌ Anti-pattern: Everything dumped in utils/
src/
├── utils/
│   ├── user-utils.js
│   ├── product-utils.js
│   ├── order-utils.js
│   ├── auth-utils.js
│   ├── validation-utils.js
│   ├── format-utils.js
│   ├── date-utils.js
│   ├── string-utils.js
│   ├── math-utils.js
│   ├── api-utils.js
│   ├── db-utils.js
│   ├── cache-utils.js
│   ├── log-utils.js
│   └── helper-utils.js  # What's in here?
├── controllers/
└── models/
```

**Problems:**
- Utils become dumping grounds
- Hard to find related code
- Unclear ownership
- Circular dependencies

**Solution:** Organize by feature, not by type.

### The "Deep Nesting" Anti-Pattern

```
// ❌ Anti-pattern: Too many nested levels
src/
├── features/
│   ├── user/
│   │   ├── management/
│   │   │   ├── authentication/
│   │   │   │   ├── login/
│   │   │   │   │   ├── handlers/
│   │   │   │   │   │   ├── controllers/
│   │   │   │   │   │   │   └── login.controller.js
│   │   │   │   │   └── services/
│   │   │   │   └── validators/
│   │   │   └── registration/
│   │   └── profile/
│   └── product/
```

**Problems:**
- Too many clicks to reach code
- Unclear structure
- Hard to navigate

**Solution:** Flatten structure, use naming conventions.

## Related Patterns

- **[Naming Conventions](./04-Naming-Conventions.md)** - Names guide organization
- **[Design Principles](./03-Design-Principles.md)** - DRY, KISS apply
- **[SOLID Principles](./02-SOLID-Principles.md)** - SRP at file level
- **[Clean Code](./01-Clean-Code.md)** - Organization affects readability