<h1>Next.js + Minio + S3 + Prisma</h1> 

<p>
    <b>This repository is a quick guide developed to assist developers there are using S3 or minIO with Next.js</b>
</p>

<h2 id="tech-stack">Tech Stack 💻</h2>

[![My Skills](https://skillicons.dev/icons?i=nodejs,react,nextjs,ts,tailwind,redis,vercel,git,github)](https://skillicons.dev)

<h2 id="project-overview">Project Overview 📋</h2>

### Getting Started

#### 1. If you're using docker, add the minIO container in the `docker-compose.yml`:

```yml
  minio:
    image: minio/minio:latest
    container_name: minio
    ports:
      - '9000:9000'
      - '9001:9001'
    environment:
      MINIO_ROOT_USER: ${S3_ACCESS_KEY_ID}
      MINIO_ROOT_PASSWORD: ${S3_SECRET_ACCESS_KEY}
    volumes:
      - ~/minio/data:/data
    command: server /data --console-address ":9001"
```

#### 2. Create a `.env` file in the root directory and add your environment variables as follows:

```env
S3_ENDPOINT="http://localhost:9000"
S3_REGION="us-west-2"
S3_ACCESS_KEY_ID="username"
S3_SECRET_ACCESS_KEY="password"
S3_BUCKET="bucket-name"
```

#### 3. Create a `s3.ts` file in the `actions.` directory and add the code as follows:

```ts
'use server'

import { auth } from '@clerk/nextjs/server'
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'
import crypto from 'crypto'

const s3 = new S3Client({
  endpoint: process.env.S3_ENDPOINT,
  region: process.env.S3_REGION,
  credentials: {
    accessKeyId: process.env.S3_ACCESS_KEY_ID!,
    secretAccessKey: process.env.S3_SECRET_ACCESS_KEY!,
  },
  /**
   * Esta flag é ESSENCIAL para o MinIO.
   * Ela força o SDK a usar o estilo de URL com caminho (ex: http://endpoint/bucket/key)
   * em vez do estilo de subdomínio (ex: http://bucket.endpoint/key), que é o padrão da AWS.
   */
  forcePathStyle: true,
})

const acceptedTypes = ['image/jpeg', 'image/png', 'image/webp']

const maxFileSize = 1024 * 1024 * 5 // 5MB

function generateFileName(bytes = 32) {
  return crypto.randomBytes(bytes).toString('hex')
}

export async function getSignedURL({
  fileType,
  fileSize,
  checksum,
}: {
  fileType: string
  fileSize: number
  checksum: string
}) {
  const session = await auth()

  if (!session) {
    return { failure: 'Not authenticated' }
  }

  if (!session.userId) {
    return { failure: 'User ID not found' }
  }

  if (!acceptedTypes.includes(fileType)) {
    return { failure: 'Invalid file type' }
  }

  if (fileSize > maxFileSize) {
    return { failure: 'File size exceeds limit' }
  }

  const putObjectCommand = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: generateFileName(),
    ContentType: fileType,
    ContentLength: fileSize,
    ChecksumSHA256: checksum,
    Metadata: {
      userId: session.userId,
    },
  })

  const signedURL = await getSignedUrl(s3, putObjectCommand, {
    expiresIn: 60,
  })

  // Here you can save the image in the db with Drizzle or Prisma or anything you want

  return { success: { url: signedURL } }
}

```

#### 4. Add `computeSHA256` in `lib` folder:

```ts
export async function computeSHA256(file: File): Promise<string> {
  const buffer = await file.arrayBuffer()
  const hashBuffer = await crypto.subtle.digest('SHA-256', buffer)
  const hashArray = Array.from(new Uint8Array(hashBuffer))
  const hashHex = hashArray.map((b) => b.toString(16).padStart(2, '0')).join('')
  return hashHex
}
```

<h2 id="contribute">Contribute 🚀</h2>

If you want to contribute, clone this repo, create your work branch and get your hands dirty!

```bash
git clone https://github.com/aridanpantoja/nextjs-minio-s3.git
```

```bash
git checkout -b feature/NAME
```

At the end, open a Pull Request explaining the problem solved or feature made, if exists, append screenshot of visual modifications and wait for the review!

### Documentations that might help

[📝 How to create a Pull Request](https://www.atlassian.com/br/git/tutorials/making-a-pull-request) |
[💾 Commit pattern](https://gist.github.com/joshbuchea/6f47e86d2510bce28f8e7f42ae84c716)

<h2 id="license">License 📃 </h2>

This project is under [MIT](./LICENSE) license
