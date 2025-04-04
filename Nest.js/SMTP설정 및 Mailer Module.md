[[Nest.js]]

# NestJS에서 SMTP 메일 서비스 구현 가이드

이 문서는 NestJS 애플리케이션에서 SMTP 메일 서비스를 구현하는 방법에 대한 종합적인 가이드입니다. Supabase 인증을 사용하는 애플리케이션에서 커스텀 이메일 발송을 구현하는 내용을 포함합니다.

## 목차

- [SMTP 서비스 구현](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#smtp-%EC%84%9C%EB%B9%84%EC%8A%A4-%EA%B5%AC%ED%98%84)
    - [필요한 패키지 설치](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#%ED%95%84%EC%9A%94%ED%95%9C-%ED%8C%A8%ED%82%A4%EC%A7%80-%EC%84%A4%EC%B9%98)
    - [모듈 설정](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#%EB%AA%A8%EB%93%88-%EC%84%A4%EC%A0%95)
    - [서비스 구현](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#%EC%84%9C%EB%B9%84%EC%8A%A4-%EA%B5%AC%ED%98%84)
    - [템플릿 생성](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#%ED%85%9C%ED%94%8C%EB%A6%BF-%EC%83%9D%EC%84%B1)
- [빌드 설정](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#%EB%B9%8C%EB%93%9C-%EC%84%A4%EC%A0%95)
- [일반적인 문제 해결](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#%EC%9D%BC%EB%B0%98%EC%A0%81%EC%9D%B8-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0)
- [Supabase 인증과 통합](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#supabase-%EC%9D%B8%EC%A6%9D%EA%B3%BC-%ED%86%B5%ED%95%A9)
- [프로젝트 구조](https://claude.ai/chat/3b76ffe9-8590-44ee-8ac7-2669bdd6cf8d#%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EA%B5%AC%EC%A1%B0)

## SMTP 서비스 구현

### 필요한 패키지 설치

```bash
npm install @nestjs-modules/mailer nodemailer handlebars
npm install -D @types/nodemailer
```

### 모듈 설정

`MailingModule`을 다음과 같이 설정합니다:

```typescript
// src/lib/mailing/mailing.module.ts
import { MailerModule } from "@nestjs-modules/mailer";
import { HandlebarsAdapter } from "@nestjs-modules/mailer/dist/adapters/handlebars.adapter";
import { Module } from "@nestjs/common";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { join } from "path";
import { MailingService } from "./mailing.service";

@Module({
  imports: [
    MailerModule.forRootAsync({
      imports: [ConfigModule], // ConfigModule을 임포트
      inject: [ConfigService],  // ConfigService를 주입
      useFactory: (configService: ConfigService) => ({
        transport: {
          host: configService.get<string>("SMTP_HOST"),
          port: configService.get<number>("SMTP_PORT"),
          auth: {
            user: configService.get<string>("SMTP_USER"),
            pass: configService.get<string>("SMTP_PASSWORD"),
          },
        },
        defaults: {
          from: `"CoinDingdong" <${configService.get<string>("SMTP_USER")}>`,
        },
        template: {
          dir: join(__dirname, "./templates"),
          adapter: new HandlebarsAdapter(),
          options: {
            strict: true,
          },
        },
        inlineCss: false, // ARM64 환경에서 문제가 발생할 경우 비활성화
      }),
    }),
  ],
  providers: [MailingService],
  exports: [MailingService],
})
export class MailingModule {}
```

### 서비스 구현

`MailingService`를 구현합니다:

```typescript
// src/lib/mailing/mailing.service.ts
import { MailerService } from "@nestjs-modules/mailer";
import { Injectable, Logger } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { CustomException } from "src/lib/exception/exceptions";
import { MailingException } from "src/lib/exception/exceptions.enum";

@Injectable()
export class MailingService {
  private readonly logger = new Logger(MailingService.name);

  constructor(
    private readonly mailerService: MailerService,
    private readonly configService: ConfigService,
  ) {}

  /**
   * 관리자 가입 확인 메일 발송
   * 
   * @returns {Promise<boolean>} 메일 발송 성공 여부
   */
  async confirmAdminRegister(): Promise<boolean> {
    try {
      await this.mailerService.sendMail({
        to: this.configService.get<string>("SUPABASE_CONFIRM_ADMIN_REGISTER_EMAIL"),
        subject: "CoinDingdong 관리자 가입 요청",
        template: "./confirmAdmin.template",
        context: {
          confirmationURL: `${this.configService.get<string>("FRONTEND_URL")}/admin/register`,
        },
      });

      this.logger.log("관리자 가입 확인 메일이 성공적으로 발송되었습니다.");
      return true;
    } catch (error) {
      this.logger.error("관리자 가입 확인 메일 발송 중 오류가 발생했습니다.", error);
      throw new CustomException(MailingException.FAILED_TO_SEND_EMAIL);
    }
  }

  /**
   * 이메일 인증 코드 발송
   * 
   * @param {string} email 수신자 이메일
   * @param {string} verificationCode 인증 코드
   * @returns {Promise<boolean>} 메일 발송 성공 여부
   */
  async sendVerificationEmail(email: string, verificationCode: string): Promise<boolean> {
    try {
      await this.mailerService.sendMail({
        to: email,
        subject: "CoinDingdong 이메일 인증",
        template: "./verification.template",
        context: {
          verificationCode,
        },
      });

      return true;
    } catch (error) {
      this.logger.error("인증 이메일 발송 중 오류가 발생했습니다.", error);
      throw new CustomException(MailingException.FAILED_TO_SEND_EMAIL);
    }
  }
}
```

### 템플릿 생성

관리자 확인용 이메일 템플릿을 생성합니다:

```handlebars
<!-- src/lib/mailing/templates/confirmAdmin.template.hbs -->
<div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; border: 1px solid #e0e0e0; border-radius: 8px;">
  <div style="text-align: center; margin-bottom: 20px;">
    <h2 style="color: #333; margin-bottom: 15px;">새로운 관리자 계정 생성 요청</h2>
    <div style="width: 100%; height: 3px; background-color: #4f46e5; margin: 20px 0;"></div>
  </div>
  
  <p style="color: #555; font-size: 16px; line-height: 1.5; margin-bottom: 20px;">
    안녕하세요, 관리자님.<br>
    새로운 관리자 계정 생성 요청이 접수되었습니다. 아래 링크를 통해 요청을 검토하고 승인해 주세요.
  </p>
  
  <div style="text-align: center; margin: 30px 0;">
    <a href="{{ confirmationURL }}" style="background-color: #4f46e5; color: white; padding: 12px 24px; text-decoration: none; border-radius: 4px; font-weight: bold; display: inline-block;">
      요청 검토하기
    </a>
  </div>
  
  <p style="color: #666; font-size: 14px; margin-top: 30px;">
    이 링크는 24시간 동안 유효합니다. 문제가 있으시면 시스템 관리자에게 문의해 주세요.
  </p>
  
  <div style="margin-top: 40px; padding-top: 20px; border-top: 1px solid #eee; text-align: center; color: #888; font-size: 12px;">
    <p>© 2025 CoinDingdong 관리자 시스템. 모든 권리 보유.</p>
    <p>본 이메일은 자동 발송되었습니다. 회신하지 마세요.</p>
  </div>
</div>
```

이메일 인증 코드용 템플릿도 생성합니다:

```handlebars
<!-- src/lib/mailing/templates/verification.template.hbs -->
<div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; border: 1px solid #e0e0e0; border-radius: 8px;">
  <div style="text-align: center; margin-bottom: 20px;">
    <h2 style="color: #333; margin-bottom: 15px;">이메일 인증 코드</h2>
    <div style="width: 100%; height: 3px; background-color: #4f46e5; margin: 20px 0;"></div>
  </div>
  
  <p style="color: #555; font-size: 16px; line-height: 1.5; margin-bottom: 20px;">
    안녕하세요,<br>
    요청하신 이메일 인증 코드입니다. 아래 코드를 입력해주세요.
  </p>
  
  <div style="text-align: center; margin: 30px 0; padding: 15px; background-color: #f9f9f9; border-radius: 4px; font-size: 24px; letter-spacing: 5px; font-weight: bold;">
    {{ verificationCode }}
  </div>
  
  <p style="color: #666; font-size: 14px; margin-top: 30px;">
    이 인증 코드는 10분 동안 유효합니다. 본인이 요청하지 않았다면 이 이메일을 무시하셔도 됩니다.
  </p>
  
  <div style="margin-top: 40px; padding-top: 20px; border-top: 1px solid #eee; text-align: center; color: #888; font-size: 12px;">
    <p>© 2025 CoinDingdong. 모든 권리 보유.</p>
    <p>본 이메일은 자동 발송되었습니다. 회신하지 마세요.</p>
  </div>
</div>
```

## 빌드 설정

템플릿 파일이 빌드 시 `dist` 폴더로 복사되도록 `nest-cli.json` 파일을 수정합니다:

```json
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "assets": [
      { "include": "lib/mailing/templates/**/*", "outDir": "dist/" }
    ],
    "watchAssets": true
  }
}
```

또는 `package.json`의 빌드 스크립트를 수정합니다:

```json
"scripts": {
  "build": "nest build && npm run copy-templates",
  "copy-templates": "cp -r src/lib/mailing/templates dist/lib/mailing/"
}
```

## 일반적인 문제 해결

### 1. CSS 인라인 처리 관련 오류

ARM64 아키텍처에서 `@css-inline/css-inline-linux-arm64-musl` 모듈을 찾을 수 없는 오류가 발생할 경우:

```
Error: Cannot find module '@css-inline/css-inline-linux-arm64-musl'
```

해결 방법:

- `inlineCss: false` 옵션을 추가하여 CSS 인라인 처리를 비활성화
- 또는 다른 기반 이미지로 Docker 설정 변경 (예: `FROM node:20-bullseye` 또는 `FROM node:20-slim`)

### 2. 템플릿 파일을 찾을 수 없는 오류

```
Error: ENOENT: no such file or directory, open '/app/dist/lib/mailing/templates/confirmAdmin.template.hbs'
```

해결 방법:

- 템플릿 파일이 올바른 위치에 있는지 확인
- 빌드 프로세스에서 템플릿 파일이 `dist` 폴더로 복사되는지 확인
- 템플릿 파일 경로를 조정 (확장자 제외 또는 상대 경로 조정)

### 3. ConfigService 관련 오류

```
TypeError: Cannot read properties of undefined (reading 'get')
```

해결 방법:

- `MailerModule.forRootAsync` 설정에 `imports: [ConfigModule]`과 `inject: [ConfigService]` 추가

## Supabase 인증과 통합

Supabase 인증 대신 커스텀 이메일 서비스를 사용하여 다음과 같은 장점을 얻을 수 있습니다:

1. 완전한 커스터마이징 가능 (디자인, 내용, 발송 시점)
2. 다양한 이메일 시나리오 지원 (관리자 확인, 사용자 인증, 알림 등)
3. 에러 처리와 로깅의 세밀한 제어

Supabase 인증과 함께 사용할 때의 워크플로우:

```typescript
/**
 * 관리자 가입 처리
 */
async adminRegister(createAdminDto: CreateAdminDto): Promise<ResponseCreateAdminDto> {
  try {
    // Supabase 인증 처리
    const decoded = this.decodeTokenService.decodeSupabaseToken(createAdminDto.code);
    const { email } = decoded;
    
    // 커스텀 이메일 발송 (Supabase 인증 대신)
    const sendingMailResult = await this.mailingService.confirmAdminRegister();

    if (sendingMailResult) {
      // 계정 생성 및 기타 로직 처리
      // ...
    }
    
    return savedAdmin;
  } catch (error) {
    // 에러 처리
  }
}
```

## 프로젝트 구조

메일링 모듈은 애플리케이션 전반에서 사용될 수 있는 유틸리티 성격의 모듈이므로 `lib` 디렉토리에 위치시키는 것이 적합합니다:

```
src/
├── lib/
│   ├── mailing/
│   │   ├── templates/
│   │   │   ├── confirmAdmin.template.hbs
│   │   │   └── verification.template.hbs
│   │   ├── mailing.module.ts
│   │   └── mailing.service.ts
│   ├── exception/
│   │   ├── exceptions.ts
│   │   └── exceptions.enum.ts
│   └── redis/
│       └── redis.service.ts
├── admin/
│   ├── admin.controller.ts
│   ├── admin.module.ts
│   └── admin.service.ts
└── app.module.ts
```

이 구조는 다음과 같은 장점이 있습니다:

1. 관심사 분리: 이메일 발송 로직이 독립적인 모듈로 분리됨
2. 재사용성: 여러 도메인(관리자, 사용자 등)에서 이메일 서비스를 재사용할 수 있음
3. 테스트 용이성: 독립적인 모듈은 단위 테스트가 더 쉬움
4. 유지보수성: 이메일 관련 변경이 필요할 때 한 곳만 수정하면 됨