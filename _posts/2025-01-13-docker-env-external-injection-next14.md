---
title: "도커 환경변수 외부 주입 with Next14"
date: 2025-01-13 08:00:00 +0900
categories: [개발]
tags: [Docker, Next14, 환경변수, CI/CD, DevOps, 빌드]
author: L.J
---

## 도커 환경변수 외부 주입 (with Next14)

### **시작하기...**

일전에 프론트엔드 배포 환경을 도커 컨테이너 기반으로 변경하는 일이 있었다. 현재는 무사히 테스트 작업이 끝나 정식 배포가 진행되었지만 여전히 아쉬운 문제가 하나 남아있었다. 바로 '도커 이미지 재사용'에 대한 문제였다.

대부분의 회사에서는 테스트 서버와 production 서버를 분리하여 운영하는 일이 많다. 그리고 당연하게도 테스트 환경과 배포 환경의 차이점이라면 주입되는 환경변수가 다르다는 정도일 것이다. ~~(환경변수 이외의 코드가 다를 경우에는 테스트의 의미가 없어져버리지 않을까....)~~ 현재 우리는 코드를 병합하면 자동으로 이미지를 생성하고 EC2에 올라가 배포를 원하는 시점에 해당 이미지로 대체되어 운영된다. 하지만 여기서 이상한 점이 있다. 결과적으로 우리의 dev 이미지와 QA 이미지, stage 이미지, prod 이미지의 차이점이라고는 환경변수 밖에 없는데, 각각의 배포 때 마다 새로운 이미지를 생성해서 배포하는 건 자원적으로도 시간적으로도 낭비이지 않을까?

그렇다. 도커의 최대 장점인 이미지 재사용을 이 '환경변수'의 차이로 인해 우리는 제대로 응용하지 못하고 있다는 문제에 봉착했다. 매번 배포 환경에 맞는 환경변수를 주입하는 대신, 그냥 빌드된 이미지 하나에 우리가 원하는 환경변수를 그때 그때 사용할 수 있도록 외부에서 주입한다면 배포 시간을 환경 개수 만큼 단축시킬 수 있고, 빌드에 필요한 자원들도 그만큼 낭비를 줄일 수 있을 것이다.

해당 블로그 포스팅은 일전에 필자가 사내에서 진행한 환경변수 외부 주입 관련 발표 자료를 다듬어 다시 작성하였다.

### **환경변수와 Next14**

애석하게도, Next14에서 환경변수를 제어하는 것은 끔찍하다. 서버 사이드 랜더링 페이지들은 그리 문제가 되지 않지만, 이놈의 클라이언트 사이드 컴포넌트가 문제다. 기본적으로 Next14는 빌드 시점에 클라이언트 사이드 컴포넌트에서 사용되는 환경변수를 미리 빌드 시점에 집어넣어 포함시키기 때문에, 런타임 시점에서 아무리 .env 파일을 변경한다 하더라도 환경변수가 제대로 변경되지 않는다.

이러한 문제를 해결하기 위해서는 미리 환경변수가 들어있는 상태가 아닌, 클라이언트 사이드 랜더링 컴포넌트들이 해당 컴포넌트의 호출 시점에 환경변수를 가져올 수 있도록 변경할 필요가 있다.

해당 문제를 해결하기 위해 여러 방법을 찾아본 결과, 카카오에서 이와 같은 문제를 논의하고 해결한 과정에 대해 내용이 작성되어 있어, 이를 기반으로 한번 작성해보려 한다.

![](https://blog.kakaocdn.net/dna/xvfMF/btsLFEDgYvn/AAAAAAAAAAAAAAAAAAAAAFRnCQN2_dONSZuO0uALXJbGURjeX2KyYlrsBprWOsBI/img.png)

우리의 목표는 다음과 같이, 하나의 빌드 이미지를 통해 각기 다른 배포가 가능하도록 만드는 것이다.

### **Next14에서 환경변수의 동작**

#### **클라이언트 사이드 환경변수 등록**

Next14 개발자라면 대부분 알고 있으리라 생각하지만, 한 번 더 짚고 간다는 의미로 적어두는 부분이다. 기본적으로 .env나 .env.local 에 접근할 때는 process.env로 접근한다. 하지만 단순히 이렇게 접근하는 방식은 클라이언트 사이드 환경변수 (태그에 직접 넣어서 사용되는 형태 등)의 경우에는 환경변수가 필수적으로 노출되다보니 Next14 보안상의 이유로 접근이 불가하다. 때문에 우리는 해당 클라이언트 사이드에서 사용되는 환경변수에 대해서는 추가적으로 next.config.js에 등록하거나 NEXT\_PUBLIC prefix를 붙여야만 한다.

이때 주의할 점은, 라우팅 방식에 따라 환경변수 사용 방식도 약간씩 달라져버린다는 것이다.

**Page router 사용중일 경우**

```javascript
// .env.local
TEST_ENV="this is test" -> server side에서 사용 가능, client side 사용 불가능
// next.config.js
env:{
  TEST_ENV=process.env.TEST_ENV -> .env.local과 같이 process.env로 접근 가능. app router와 page router에서 서로 다르게 동작
}
```

page router를 사용중일 경우, 위와 같이 next.config.js에 해당 환경변수를 사용하겠다고 선언하는 것을 통해 client side에서도 환경변수가 접근 가능하게 만들 수 있다. 또한 NEXT_PUBLIC prefix를 .env에 선언된 환경변수들 중 client side에서도 사용할 필요가 있는 환경변수에 대하여 붙여두면, 굳이 next.config.js에 등록하지 않아도 클라이언트 사이드에서 사용할 수 있게 된다.

**App router 사용중인 경우**

이 경우, 위의 방법 중 next.config.js에 등록하여 사용하는 방법은 불가하고, NEXT_PUBLIC으로 작성한 환경변수만 클라이언트 사이드에서 접근이 허용된다.

#### **buildtime 환경 변수, runtime 환경 변수**

앞서 잠깐 언급하기도 했지만 다시 클라이언트 사이드 환경변수와 서버 사이드 환경변수의 동작에 대해 buildtime과 runtime에 어떤 차이를 보이는지 정리해둔다.

### **어떻게 해결해야 할까**

결국 우리가 원하는 것은, 빌드 타임이 아닌 런타임에 원하는 환경변수가 세팅되어 도커 이미지에서 실행시킬 수 있어야 한다는 것이다. 이를 위해서 다양한 자료를 찾아본 결과, 가장 필자가 원하는 방식과 유사한 방법으로 해당 문제를 해결한 글을 찾을 수 있었다.

<https://fe-developers.kakaoent.com/2022/220505-runtime-environment/>

해당 방법은 클라이언트 환경변수를 런타임에 원하는 .env 파일을 읽어들여 사용할 수 있도록 한다. 하지만 문제가 있다. 해당 환경변수 URL 방식을 사용하게 될 경우, 환경변수가 통째로 __ENV을 통해 network에 노출된다는 치명적인 문제가 있다. 클라이언트 사이드 환경변수를 해결하려다 서버 사이드 환경변수까지 노출시키는 것은 우리가 원하는 결과가 아니다.

그래서 필자는 클라이언트 환경변수에 대하여 NEXT_PUBLIC prefix를 사용하고, 해당 prefix가 있을 경우에만 __ENV에 넣고, 해당 환경변수들을 한 번 더 암호화하여 안전하게 사용할 수 있도록 해보았다. ~~(물론, 클라이언트 환경변수들이 노출되는 건 어쩔 수 없는 부분이다. 때문에 서버 쪽에 치명적인 키와 같은 정보들이 클라이언트 사이드에서 사용되지 않도록 주의가 필요하다)~~

#### **코드 작성**

utils와 같은 폴더에 해당 코드들을 관리하는 것이 좋다. 도커 이미지를 실행할 때, 환경변수로 develop이나 prod를 넘기면, .env.develop 혹은 .env.prod 환경변수 파일을 사용하여 필요한 구성을 해주는 방식이다. **이때 환경변수 파일명을 development나 production으로 해서는 안된다. 두 이름은 실제 환경변수 파일에서 지원하는 이름이다 보니, 원치 않는 동작이 수행될 수 있다.**

**cli.js**

```javascript
import yargs from "yargs/yargs";
import { hideBin } from "yargs/helpers";
import parseDotenv from "./parsedotenv.mjs";
import writeEnv from "./writeEnv.mjs";
import copyEnv from "./copyEnv.mjs";

yargs(hideBin(process.argv))
  .command(
    "next-env",
    "Create Next.js runtime environment js",
    function builder(y) {
      return y.option("env", {
        alias: "e",
        type: "string",
        description: "Environment name(ex: alpha, dev, staging, real)",
      });
    },
    async function handler(args) {
      const appEnv = args.e || args.env || "develop";
      const parsedEnv = await parseDotenv(appEnv); // dotenv 파싱
      writeEnv(parsedEnv); // 환경 변수 스크립트 파일 생성
      await copyEnv(appEnv); // .env 파일 복사
      return parsedEnv;
    },
  )
  .parse();
```

해당 코드는 런타임에 동작하며, 사용중인 .env.develop 혹은 .env.prod를 기반으로 .env파일과 __ENV.js 파일을 생성한다.

**copyEnv.js**

```javascript
import { findUp } from 'find-up';
import fs from 'fs';

export default async function copyEnv(appEnv) {
  // 파싱 대상 파일은 '.env'파일로 복사
  const envFilePath = await findUp(`.env.${appEnv}`);
  const dotenvFilePath = `${fs.realpathSync(process.cwd())}/.env`;
  fs.copyFileSync(envFilePath, dotenvFilePath);
}
```

.env 파일을 생성하는 코드

**env.js**

```javascript
import CryptoJS from 'crypto-js';

const decrypt = (encrypted) => {
  try {
    const secret_key = process.env.NEXT_PUBLIC_CRYPTO_SECRET_KEY;
    if (!secret_key) {
      console.log('No Secret Key. decript');
      return '';
    }
    const decrypted_bytes = CryptoJS.AES.decrypt(encrypted, secret_key);
    const decrypted = decrypted_bytes.toString(CryptoJS.enc.Utf8);
    return decrypted;
  } catch (e) {
    console.log('Decryption error occur : ', e);
    return '';
  }
};

export default function env(envName) {
  if (!envName) {
    return undefined;
  }
  if (typeof window !== 'undefined' && window.__ENV) {
    const envVar = window.__ENV[envName];
    const decrypted = decrypt(envVar);
    return decrypted;
  } else {
    const envVar = process.env[envName];
    return envVar;
  }
}
```

클라이언트 사이드 환경변수를 사용할 때 process.env.변수명 이 아닌, env(변수명)으로 변경한다.

**parsedotenv.js**

```javascript
import Dotenv from 'dotenv';
import { findUp } from 'find-up';

export default async function parseDotenv(appEnv) {
  // dotenv 파싱
  const envFilePath = await findUp(`.env.${appEnv}`);
  const parsedEnv = Dotenv.config({ path: envFilePath }).parsed || {};
  return parsedEnv;
}
```

.env 파일 경로를 전달한다.

**writeEnv.js**

```javascript
import fs from 'fs';
import path from 'path';
import CryptoJS from 'crypto-js';

const encrypt = (payload) => {
  try {
    const secret_key = process.env.NEXT_PUBLIC_CRYPTO_SECRET_KEY;
    if (!secret_key) {
      console.log('No Secret Key. encrypt');
      return null;
    }
    const encrypted = CryptoJS.AES.encrypt(payload, secret_key).toString();
    return encrypted;
  } catch (e) {
    console.log('Encryption error occur : ', e);
    return null;
  }
};

export default function writeEnv(parsedEnv) {
  // __ENV.js 파일 경로 설정
  const scriptFilePath = path.join(process.cwd(), 'public', '__ENV.js');
  // 환경 변수 객체 초기화
  const envObject = {};
  // parsedEnv 객체에서 NEXT_PUBLIC 접두사가 있는 항목만 추가
  Object.keys(parsedEnv).forEach((key) => {
    if (key.startsWith('NEXT_PUBLIC') && key.includes('CRYPTO_SECRET_KEY') === false) {
      envObject[key] = encrypt(parsedEnv[key].toString());
    }
  });
  // __ENV.js 파일에 쓰기
  const scriptContent = `window.__ENV = ${JSON.stringify(
    envObject,
    null,
    2,
  )};`;
  fs.writeFileSync(scriptFilePath, scriptContent);
}
```

__ENV.js 파일을 생성할 때, NEXT_PUBLIC prefix가 붙어있는 환경변수만 __ENV.js에 밀어넣는다. 이때 NEXT_PUBLIC_CRYPTO_SECRET_KEY는 __ENV.js에 포함되지 않게 했다. 위와 같이 crypto 라이브러리를 통해 노출되는 키들에 대하여 암호화를 수행하고 추후 사용할 때 env에서 복호화가 된다.

**_document.tsx**

```tsx
<Head>
  <script src="/__ENV.js" />
</Head>
```

__ENV.js가 랜더링 시 미리 로드되게 지정한다.

**package.json**

```json
"dev": "node ./src/utils/cli.mjs next-env --env=${APP_ENV:-develop} && next dev",
"start": "node ./src/utils/cli.mjs next-env --env=${APP_ENV:-develop} && next start",
```

dockerfile에서 전달받은 환경변수를 넣어 develop 혹은 prod 환경변수를 구성한다.

**dockerfile**

```yaml
version: '3.8'
services:
  app:
    image: nextjs
    build:
      context: ./
      target: runner
      dockerfile: Dockerfile
    environment:
      - APP_ENV=develop
    expose:
      - "3000"
```

APP_ENV를 통해서 원하는 환경을 세팅할 수 있다.

### **마무리**

해당 방법을 통해 우리는 도커 이미지를 환경변수에 따라 여러번 빌드할 필요 없이 하나의 이미지를 재사용하여 세 가지 서로 다른 환경을 세팅할 수 있도록 해보았다. 클라이언트 사이드에 노출되는 환경변수는 대부분 노출되어도 문제없는 정보이거나, 키로 사용되어도 도메인이 등록되어야만 정상적으로 동작하는 것들이 대부분이다보니 크게 신경쓰지 않아도 되는 부분이라 생각할 수 있지만, 실수로라도 서버 사이드의 중요 키가 노출될 경우 치명적인 보안 문제가 될 수 있기 때문에 늘 조심하는 것이 좋다.

추가적으로 __ENV.js에 대해서도 너무 대놓고 환경변수임을 티를 내는 이름이기 때문에, 눈에 띄지 않는 파일명으로 교체하는 것도 추천한다. 또한 이번 기회에 불필요하게 서버 사이드 환경변수가 노출되고 있지는 않은지 확인을 할 수 있어서 나름 다양한 방면에서 코드를 개선하고 점검할 수 있는 시간이었다.