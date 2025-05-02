## ✍️ 블로그 글 작성 방법

1. 글은 `content/blog/` 폴더에 작성합니다.
2. 각 글은 **하위 폴더**로 구분되며, **`index.md`** 파일을 포함해야 합니다.
3. 파일 구조 예시:
    ```
    content/blog/
    ├── my-first-post/
    │   └── index.md
    └── another-post/
        └── index.md
    ```

4. `index.md`의 **YAML frontmatter** 예시:

    ```markdown
    ---
    title: "포스트 제목"
    date: "2025-05-02"
    description: "포스트 설명 (선택)"
    ---

    여기에 본문 내용을 작성합니다. 마크다운 문법을 사용할 수 있습니다.

    ## 예시
    - 리스트
    - 코드 블록
    - 이미지 등
    ```

5. 이미지 사용 방법:
    - 같은 폴더 내에 이미지를 넣고 다음처럼 참조:
      ```markdown
      ![설명](./image.png)
      ```

6. 작성 후 로컬에서 확인:
    ```bash
    npm run develop
    ```

7. 커밋 및 푸시:
    ```bash
    git add .
    git commit -m "feat: 블로그 글 작성 - my-first-post"
    git push
    ```

8. GitHub Pages 배포가 자동으로 실행됩니다 (`deploy.yml` 참고).
