# 개인 기술 블로그

Material for MkDocs + GitHub Pages로 운영하는 문서 전용 프로젝트입니다.
이 폴더 자체를 별도 GitHub 저장소의 최상위로 사용합니다. 상위 실습 프로젝트는 올리지 않습니다.

## 로컬에서 보기

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m mkdocs serve --dev-addr 127.0.0.1:8000
```

브라우저에서 http://127.0.0.1:8000 을 엽니다.
빌드 검사는 `python -m mkdocs build --strict`로 실행합니다.

## GitHub Pages

- 블로그: https://heesangjin.github.io/tech-blog/
- 저장소: https://github.com/HeesangJin/tech-blog
- 게시 방식: GitHub Actions (`.github/workflows/deploy.yml`)

`main`에 push하면 MkDocs 검사·빌드 후 GitHub Pages에 배포합니다.
저장소의 **Settings → Pages → Source**는 **GitHub Actions**입니다.
실행 상태는 저장소의 **Actions → Deploy technical blog**에서 확인합니다.

```bash
git add docs mkdocs.yml
git commit -m "Update blog post"
git push
```

## 글 추가하기

- 글은 `docs/<주제>/<글이름>.md`, 이미지는 해당 주제의 `assets/`에 넣습니다.
- `mkdocs.yml`의 `nav`와 주제별 `index.md`에 링크를 추가합니다.
- `main`에 push하면 GitHub Actions가 검사하고 배포합니다.
- 테마 기본값으로 하단의 Material for MkDocs 출처가 표시됩니다.
- 이 사이트는 주제별 문서형 블로그입니다. 날짜별 피드용 `blog` 플러그인은 사용하지 않습니다.
- 현재 메뉴에 포함된 글은 배포하면 공개됩니다.

## 포함 범위

`docs/`의 글·이미지, 사이트 설정, 의존성 목록, 배포 워크플로만 있습니다.
실습 Python 모듈, 노트북, 데이터, 원시 검증 로그와 인프라 배포 파일은 포함하지 않습니다.
본문의 코드 블록은 설명을 위한 발췌 예제입니다.

## 실습 글의 화면과 코드

실제 대시보드 캡처 9장과 Python 핵심 예시를 포함했습니다.
이미지를 클릭하면 원본 크기로 볼 수 있습니다.
화면을 준비하기 위해 학습·추론을 재실행하지 않고 보존된 성공 Run을 열었습니다.
