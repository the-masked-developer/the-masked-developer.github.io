## Prequisition

- Node 12

```bash
sudo n 12
```

## Quick Start

### 클론

```bash
git clone https://github.com/the-masked-developer/the-masked-developer.github.io.git
```

### 브랜치 변경(writing)

> 이미지 기본 브랜치가 `writing`으로 첫 클론시 할 필요 없음

```bash
git checkout -b writing
```

### 설치

```bash
npm install
```

### 문서 업로드

`/source/_posts` 경로에 마크다운(`.md`) 파일 업로드 (형식은 이미 올라온 해당 경로의 마크다운 참조)

### 로컬 테스트

```bash
npm run serve
```

### 배포

Github Action 사용으로 `git push`시 자동으로 배포되어 [https://the-masked-developer.github.io/](https://the-masked-developer.github.io/) 주소에서 확인 가능
