---
layout: post
title: "Claude AI와 GitHub 블로그 자동화하기"
date: 2025-06-14 15:00:00 +0900
categories: [Tech, Automation]
tags: [claude, github, mcp, automation, jekyll]
---

# Claude AI로 GitHub 블로그를 자동화해보자! 🤖

최근 Claude AI의 MCP(Model Context Protocol) 기능을 활용해서 GitHub 블로그를 자동화하는 작업을 진행했습니다. 그 과정에서 겪었던 문제들과 해결 방법을 정리해봤어요.

## 🔧 MCP 서버 설정하기

### 1. 파일 시스템 연결
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "C:\\Users\\username\\Desktop\\test"
      ]
    }
  }
}
```

### 2. GitHub 연결
```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your_token_here"
      }
    }
  }
}
```

**설정 파일 위치:**
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`

## 🚨 Jekyll 블로그 문제 해결

### 가장 큰 문제: baseurl 설정
```yaml
# ❌ 잘못된 설정
baseurl: "/username"

# ✅ 올바른 설정 (username.github.io 저장소)
baseurl: ""
```

GitHub Pages에서 `username.github.io` 저장소는 baseurl이 빈 값이어야 합니다!

### 포스트가 홈에 안 나타나는 문제
- **날짜 확인**: Jekyll은 미래 날짜 포스트를 숨김
- **layout 명시**: `layout: post` 반드시 추가
- **핀 기능**: `pin: true`로 중요한 포스트 상단 고정

### GitHub Pages 호환성
```yaml
# _config.yml에서 plugins 섹션 제거
# GitHub Pages는 특정 플러그인만 지원

# Gemfile 설정
gem "github-pages", group: :jekyll_plugins
gem "jekyll-theme-chirpy"
```

## 💡 Claude와 GitHub 연동 꿀팁

### Personal Access Token 생성
1. GitHub → Settings → Developer settings
2. Personal access tokens → Tokens (classic)
3. **필수 권한**: `repo`, `user`, `gist`, `notifications`

### MCP 연결 확인 방법
```
Claude에서 "내 GitHub 저장소 목록을 보여줘" 명령어로 테스트
```

### 자동 포스트 생성
Claude가 직접 마크다운 파일을 `_posts/` 폴더에 생성 가능!

## 🔄 Git 되돌리기 꿀팁

### 빌드 실패 시 이전 커밋으로 복구
```bash
# 2개 전 커밋으로 되돌리기
git reset --hard HEAD~2
git push --force-with-lease origin master

# 특정 커밋으로 되돌리기
git reset --hard [커밋SHA]
git push --force-with-lease origin master
```

### 안전한 되돌리기
```bash
# 변경사항 유지하면서 되돌리기
git reset --soft HEAD~2
git commit -m "Revert to stable version"
git push origin master
```

## 📝 배운 점들

1. **MCP는 정말 강력함**: 파일 시스템과 GitHub을 동시에 제어 가능
2. **Jekyll 설정의 중요성**: 작은 설정 하나로 전체가 망가질 수 있음
3. **빌드 오류 대응**: Git reset으로 빠르게 복구하고 천천히 재시도
4. **테스트의 중요성**: 간단한 포스트부터 시작해서 단계적으로 확장

## 🎯 결론

Claude AI와 GitHub을 연동하면 블로그 관리가 정말 편해집니다. 포스트 작성부터 배포까지 모든 과정을 자동화할 수 있어요!

다만 초기 설정에서 삽질을 좀 할 수 있으니, 차근차근 설정을 확인하면서 진행하는 것이 중요합니다.

---

*이 포스트도 Claude AI가 직접 생성했습니다! 🎉*
