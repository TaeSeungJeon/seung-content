---
title: "👏👏중간프로젝트 '씨네마-톡' 리뷰"
date: "2026-04-07"
description: "Servlet MVC 지옥의 맛 찍먹 ver 1."
category: "Dev"
---

# 개발 환경 & ERD 보고 가실게요~


![Image](https://github.com/user-attachments/assets/480e6514-b1a1-4523-baed-c6a0e18363eb)

---


### 게시판(Board) 관련 패키지 간략 구조

```
Controller/Board/  ← 요청 처리 (각 기능별 Controller 분리)
Service/Board/     ← 비즈니스 로직
DAO/Board/         ← DB 접근 (MyBatis SqlSession 직접 관리)
DTO/Board/         ← 데이터 전달 객체
mappers/Board/     ← MyBatis XML Mapper (SQL 분리)
SQL/Board/         ← DDL (테이블 생성 스크립트)
```

---

# 1. 게시글 작성 (BoardOkController)

![Image](https://github.com/user-attachments/assets/996fac9f-dcf8-4fcc-b2c6-39581edf7cf8)

### 구현 내용

- 로그인 세션 검증 → 비로그인 시 로그인 페이지 리다이렉트
(비회원 글쓰기 불가)
- 제목/내용 공백 유효성 검사
- `boardType` 파라미터로 게시판 분기 (자유/영화/공지/문의)
- 영화 연동: `movieId` 전달 시 DB에서 `movieTitle` 조회 후 비정규화 저장
- 게시글 등록 성공 후 타입별 URL로 리다이렉트

## 1-1. 게시글 모달 (파일 첨부 / 링크 첨부 / 글꼴 옵션)

### 구현 방식

- 글쓰기를 페이지 이동 없이 **모달로 처리** → UX 흐름 유지
- 에디터 글꼴 옵션 지원 (굵기, 크기 등 서식)
- **링크 미리보기**: 글 저장 전 URL 입력 시 실시간으로 OG 태그 카드 렌더링
- **파일 첨부**: `ADD_FILE` 테이블에 별도 저장, 게시글과 `(boardId, boardType)` FK로 연결
- UUID로 파일명 충돌 방지

### 핵심 코드

#### `BoardOkController.java`

```java
int movieIdInt = (movieId == null || movieId.trim().isEmpty()) ? -1 : Integer.parseInt(movieId);
String movieTitle = boardService.getMovieTitleforBoard(movieIdInt);
bdto.setMovieId(movieIdInt);
bdto.setMovieTitle(movieTitle);
```

#### `BoardMapper.xml`

```xml
<insert id="boardIn" parameterType="DTO.Board.BoardDTO">
    <selectKey keyProperty="boardId" resultType="int" order="BEFORE">
        SELECT boardIdseq.nextval FROM dual
    </selectKey>
    INSERT INTO BOARD (boardId, boardType, boardTitle, boardContent, boardName, boardDate, memNo, linkUrl
    <if test="movieId != -1">, movieId, movieTitle</if>)
    VALUES (#{boardId}, #{boardType}, #{boardTitle}, #{boardContent}, #{boardName}, SYSDATE, #{memNo}, #{linkUrl}
    <if test="movieId != -1">, #{movieId}, #{movieTitle}</if>)
</insert>

<!--
- MyBatis `<selectKey>`로 INSERT 전 시퀀스 채번
- `<if test="movieId != -1">` 동적 SQL → 영화 미선택 시 null 저장
-->
```

#### `BoardOkController.java` — 파일 업로드 처리

```jsx
Collection<Part> parts = request.getParts();
for (Part part : parts) {
    if (part.getName().equals("uploadFile") && part.getSize() > 0) {
        String originalFileName = Paths.get(part.getSubmittedFileName()).getFileName().toString();
        String savedFileName = UUID.randomUUID().toString() + "_" + originalFileName;
        // UUID로 파일명 충돌 방지
        String savePath = uploadDir + File.separator + savedFileName;
        part.write(savePath);
        // AddFileDTO → DB 저장
    }
}
```

### 링크 미리보기 구조

#### `LinkPreviewController.java` — SSRF 방지 포함

```jsx
// 로컬/사설망 IP 차단
InetAddress addr = InetAddress.getByName(host);
if (addr.isAnyLocalAddress() || addr.isLoopbackAddress() || addr.isSiteLocalAddress()) {
    throw new IllegalArgumentException("로컬/사설망 접근 차단");
}
// OG 태그 파싱
String title       = pickMeta(doc, "meta[property=og:title]", "content");
String description = pickMeta(doc, "meta[property=og:description]", "content");
String image       = pickMeta(doc, "meta[property=og:image]", "content");
String siteName    = pickMeta(doc, "meta[property=og:site_name]", "content");
```

### 영화 검색 자동완성

```jsx
// 300ms debounce 적용 → 입력마다 요청 방지
searchTimer = setTimeout(() => {
    $.ajax({ url: 'searchMovie.do', data: { "search-words": query } ... });
}, 300);

// 키보드 방향키로 검색 결과 탐색
if (e.keyCode == 40) { currentFocus++; addActive(items); }  // 아래
if (e.keyCode == 38) { currentFocus--; addActive(items); }  // 위
if (e.keyCode == 13) { items[currentFocus].click(); }       // 선택
```

---

# 2. 게시글 목록 & 페이지네이션 (BoardListController)

![Image](https://github.com/user-attachments/assets/23e61ac4-55ea-46b3-bd27-112d351c3875)

### 구현 내용

- `filter` 파라미터로 전체/자유/영화 게시판 분기
- **페이지네이션**: `startRow`, `endRow` 계산 후 MyBatis에 전달
- 10페이지 블록 단위 페이지 네비게이션
- 목록에서 각 게시글의 댓글 수 실시간 조회
- 사이드바 일간/주간/월간 인기글 동시 조회
- 최신 공지사항 1건 상단 고정

### 핵심 코드

#### `BoardListController.java`

```java
int startRow = (page - 1) * limit + 1;
int endRow   = startRow + limit - 1;

if ("free".equals(filter)) {
    totalCount = service.getBoardCountByType(1);
    list = service.boardListPageByType(1, startRow, endRow);
} else if ("hot".equals(filter)) {
    totalCount = service.getBoardCountByType(2);
    list = service.boardListPageByType(2, startRow, endRow);
} else {
    totalCount = service.getBoardCount();
    list = service.boardListPage(startRow, endRow);
}
// 10페이지 블록
int startPage = ((page - 1) / 10) * 10 + 1;
int endPage   = Math.min(startPage + 9, maxPage);
```

#### `BoardListController.java` — 페이지 계산

```jsx
int startRow = (page - 1) * limit + 1;
int endRow   = startRow + limit - 1;

// 10페이지 블록
int startPage = ((page - 1) / 10) * 10 + 1;
int endPage   = Math.min(startPage + 9, maxPage);
```

#### `BoardMapper.xml` — Oracle ROWNUM 페이징

```jsx
<select id="boardListPage">
    SELECT * FROM (
        SELECT ROWNUM rnum, b.*,
               (SELECT COUNT(*) FROM BOARD_LIKE l
                WHERE l.boardId = b.boardId AND l.boardType = b.boardType) AS likeCount
        FROM (SELECT * FROM BOARD ORDER BY boardId DESC) b
        WHERE ROWNUM &lt;= #{endRow}
    )
    WHERE rnum >= #{startRow}
</select>
```

#### `BoardSearchMapper.xml` — 검색 결과 페이징 (동적 SQL + 페이징 중첩)

```jsx
<select id="searchBoardByTitleAndCont">
    SELECT * FROM (
        SELECT ROWNUM rnum, b.*
        FROM (SELECT b.*,
                     (SELECT COUNT(*) FROM BOARD_LIKE ...) AS likeCount
              FROM BOARD b WHERE 1=1
              <if test="type != 0">AND BOARDTYPE = #{type}</if>
              AND (
                <foreach collection="wordList" item="word" separator="OR">
                    UPPER(BOARDTITLE) LIKE '%'||UPPER(#{word})||'%'
                    OR UPPER(BOARDCONTENT) LIKE '%'||UPPER(#{word})||'%'
                </foreach>
              )
              ORDER BY boardId DESC) b
        WHERE ROWNUM &lt;= #{endrow}
    )
    WHERE rnum >= #{startrow}
</select>

/*
 - `<foreach>` 로 공백 분리 다중 단어 검색 지원 (`OR` 연결)
 - `UPPER()` 적용 → 대소문자 무관 검색
*/
```

---

## 2-1 . 게시글 검색 (BoardSearchController)

### 구현 내용

- 검색어 2글자 미만 유효성 검사
- `searchOption`으로 제목/내용/작성자 검색 분기
- **영화 ID 기반 검색** 지원 (`movieId` 파라미터)
- 검색 결과에도 동일한 페이지네이션 적용
- `filter`에 따라 게시판별 검색 범위 제한

### 핵심 코드

#### `BoardSearchController.java`

```java
if (movieId != 0) {
    totalCount = searchService.getBoardCountByMovieId(movieId);
    list = searchService.boardListPageByMovieId(movieId, startRow, endRow);
} else {
    totalCount = searchService.getBoardCountByTypeAndWord(type, searchWords, searchOption);
    list = searchService.boardListPageByTypeAndWord(type, startRow, endRow, searchWords, searchOption);
}
```

---

# 3. 게시글 상세 & 링크 미리보기 (PostDetailController)

![Image](https://github.com/user-attachments/assets/93bef593-28cd-4e9b-bf98-8057768a0f63)

### 구현 내용

- 조회수 자동 증가 (`updateReadCount` → 무조건 commit)
- 게시글 내 URL 자동 감지 → **OG 태그 기반 링크 미리보기** 생성
- 사이드바: 실시간 인기글 + 일/주/월간 인기글 동시 표시
- 첨부파일 목록 조회
- 댓글 목록 + 각 댓글의 좋아요 수/여부 포함 조회
- 댓글 작성자의 프로필 사진, 회원 등급(memRole) 표시

### 핵심 코드

#### `BoardServiceImpl.java` - 링크 미리보기

```java
private LinkPreviewDTO fetchPreview(String url) {
    Document doc = Jsoup.connect(url)
            .userAgent("Mozilla/5.0")
            .timeout(3000).get();
    String title = doc.select("meta[property=og:title]").attr("content");
    String desc  = doc.select("meta[property=og:description]").attr("content");
    String image = doc.select("meta[property=og:image]").attr("content");
    if (isBlank(title) && isBlank(desc) && isBlank(image)) return null;
    return new LinkPreviewDTO(url, title, desc, image);
}
```

#### `LinkPreviewController.java` - SSRF 방지 로직

```java
private void validateUrl(String url) throws Exception {
    InetAddress addr = InetAddress.getByName(host);
    if (addr.isAnyLocalAddress() || addr.isLoopbackAddress() || addr.isSiteLocalAddress()) {
        throw new IllegalArgumentException("로컬/사설망 접근 차단");
    }
}
```

## 3-1. 게시글 좋아요 (BoardLikeToggleController)

### 구현 내용

- 비로그인 시 `"LOGIN_REQUIRED"` 문자열 응답 → JS에서 처리
- 토글 방식: 좋아요 존재 시 취소, 없으면 추가
- 변경 후 현재 좋아요 수 숫자만 응답 (경량 응답)

### 핵심 코드

#### `BoardLikeToggleController.java`

```java
int likeCount = service.toggleBoardLike(boardId, boardType, memNo);
out.print(likeCount);  // 숫자만 반환
```

#### `BoardServiceImpl.java`

```java
public int toggleBoardLike(int boardId, int boardType, int memNo) {
    int liked = bdao.isBoardLiked(boardId, boardType, memNo);
    if (liked > 0) {
        bdao.deleteBoardLike(boardId, boardType, memNo);
    } else {
        bdao.insertBoardLike(boardId, boardType, memNo);
    }
    return bdao.getBoardLikeCount(boardId, boardType);
}
```

## 3-2. 실시간 인기글 (HotBoardController), 사이드 바

### 구현 내용

- JSON 응답 API (별도 라이브러리 없이 `StringBuilder`로 직접 직렬화)
- 좋아요 수 → 조회수 순 정렬 (Oracle 서브쿼리 + ROWNUM)
- 일/주/월간 인기글도 기간 파라미터 하나로 처리

`PostDetailController.java` — 상세 페이지 진입 시 사이드바 데이터 동시 로드

```jsx
// 실시간 인기글
List<BoardDTO> hotList = service.hotBoardList(10);
request.setAttribute("hotList", hotList);

// 기간별 인기글 3종 동시 조회
List<BoardDTO> dailyPopularList   = service.getPopularBoardList("daily",   10);
List<BoardDTO> weeklyPopularList  = service.getPopularBoardList("weekly",  10);
List<BoardDTO> monthlyPopularList = service.getPopularBoardList("monthly", 10);

request.setAttribute("dailyPopularList",   dailyPopularList);
request.setAttribute("weeklyPopularList",  weeklyPopularList);
request.setAttribute("monthlyPopularList", monthlyPopularList);
```

`HotBoardController.java` — JSON API (사이드바 비동기 갱신용)

```jsx
// 외부 라이브러리 없이 StringBuilder로 직접 직렬화
json.append("{\"items\":[");
for (int i = 0; i < list.size(); i++) {
    BoardDTO dto = list.get(i);
    json.append("{")
        .append("\"rank\":").append(i + 1).append(",")
        .append("\"boardId\":").append(dto.getBoardId()).append(",")
        .append("\"title\":\"").append(dto.getBoardTitle().replace("\"", "\\\"")).append("\",")
        .append("\"likeCount\":").append(dto.getLikeCount()).append(",")
        .append("\"readCount\":").append(dto.getBoardViewCount())
        .append("}");
}
json.append("]}");
```

`BoardMapper.xml`

```xml
<!-- 실시간 인기글 -->
<select id="hotBoardList">
    SELECT * FROM (
        SELECT b.BOARDID, b.BOARDTYPE, b.BOARDTITLE, b.BOARDVIEWCOUNT,
               (SELECT COUNT(*) FROM BOARD_LIKE l
                WHERE l.BOARDID = b.BOARDID AND l.BOARDTYPE = b.BOARDTYPE) AS likeCount
        FROM BOARD b WHERE b.BOARDTYPE IN (1,2)
        ORDER BY likeCount DESC, NVL(b.BOARDVIEWCOUNT, 0) DESC, b.BOARDID DESC
    ) WHERE ROWNUM <= #{limit}
</select>

<!-- 기간별 인기글 -->
<select id="getPopularBoardList">
    SELECT * FROM (
        ...
        WHERE (#{period} = 'daily'   AND b.BOARDDATE >= SYSDATE - 1)  OR
              (#{period} = 'weekly'  AND b.BOARDDATE >= SYSDATE - 7)  OR
              (#{period} = 'monthly' AND b.BOARDDATE >= SYSDATE - 30)
        ORDER BY likeCount DESC, b.BOARDVIEWCOUNT DESC
    ) WHERE ROWNUM <= 10
</select>
```

---

# 4. 게시글 수정/삭제 
(BoardUpdateOkController/BoardDeleteController)

![Image](https://github.com/user-attachments/assets/d360d61c-ba24-4179-9061-0c7ebf7ed741)

### 구현 내용

- 수정/삭제 모두 **서버에서 작성자 본인 검증** (세션 `memNo` vs DB `memNo` 비교)
- MyBatis `<choose><when>` 동적 SQL → movieId 있을 때만 영화 정보 UPDATE
- 삭제 시 `BOARD_LIKE`, `COMMENTS`는 `ON DELETE CASCADE`로 자동 삭제

#### `BoardMapper.xml`

```xml
<update id="updateBoardContent">
    UPDATE BOARD SET BOARDTITLE = #{boardTitle}, BOARDCONTENT = #{boardContent},
    <choose>
        <when test="movieId != -1">MOVIEID = #{movieId}, MOVIETITLE = #{movieTitle}</when>
        <otherwise>MOVIEID = NULL, MOVIETITLE = NULL</otherwise>
    </choose>
    WHERE BOARDID = #{boardId}
</update>
```

#### `BoardUpdateOkController.java` — 작성자 검증

```jsx
BoardDTO bdto = boardService.getBoardCont(boardId);
if (bdto == null || !bdto.getMemNo().equals(loginMemNo)) {
    forward.setPath("freeBoard.do");
    forward.setRedirect(true);
    return forward;
}
boardService.deleteBoard(boardId);
```

#### `BoardUpdateController.java` — 수정 폼 진입 시에도 권한 검증

```jsx
// 수정 폼 진입 시에도 서버에서 권한 검증
BoardDTO board = boardService.getBoardCont(boardId);

if (board == null || !board.getMemNo().equals(loginMemNo)) {
    forward.setPath("freeBoard.do");
    forward.setRedirect(true);
    return forward;
}
// 검증 통과 시만 기존 내용을 request에 담아 수정 폼으로 forward
request.setAttribute("board", board);
forward.setPath("/WEB-INF/views/board/boardUpdate.jsp");
forward.setRedirect(false);  
```

#### `BoardMapper.xml` — 수정 시 영화 연결 해제 처리

```jsx
<update id="updateBoardContent">
    UPDATE BOARD SET BOARDTITLE = #{boardTitle}, BOARDCONTENT = #{boardContent},
    <choose>
        <when test="movieId != -1">
            MOVIEID = #{movieId}, MOVIETITLE = #{movieTitle}
        </when>
        <otherwise>
            MOVIEID = NULL, MOVIETITLE = NULL   <!-- 영화 연결 해제 -->
        </otherwise>
    </choose>
    WHERE BOARDID = #{boardId}
</update>
```

---

# 5. 사이드바 & 홈 하단 인기글/최근글 노출

![Image](https://github.com/user-attachments/assets/9a360334-9325-4be0-bf2d-bd24a2bfff9f)

### 구현 방식

- **실시간 인기글**: 좋아요 수 → 조회수 순 정렬, JSON API로 별도 제공
- **기간별 인기글**: daily/weekly/monthly 단일 파라미터로 분기 (`SYSDATE - 1/7/30`)
- **최근 게시글**: 최신 등록순 TOP N
- 모든 리스트는 상세 페이지 진입 시에도 동시 조회 → 사이드바 항상 최신 상태 유지

---

# 6. 댓글 시스템 (CommentsOkController/CommentsDAO)

![Image](https://github.com/user-attachments/assets/d38f8750-7165-496b-bcbd-af344d1ac13d)

### 구현 내용

- 댓글 + 대댓글 (자기참조 FK `parentBoardId`)
- 댓글별 좋아요 토글 (COMMENTS_LIKE 테이블)
- `commentsListWithLike()`: 댓글 목록 조회 시 현재 사용자의 좋아요 여부 포함 반환
- 댓글 삭제 시 `commentsDeleteTree()` → 대댓글 연쇄 삭제
- 댓글 작성자 프로필 사진 & 회원 등급 표시
- **자기참조 FK** → 대댓글(계층 댓글) 구조
- `ON DELETE CASCADE` → 부모 댓글 삭제 시 자식 대댓글 연쇄 삭제

### DTO 구조

`CommentsDTO.java`

```java
private Integer parentBoardId;  // 대댓글용, null = 최상위 댓글
private Integer parentBoardNo;
private Integer likeCount;      // 댓글 좋아요 수
private Boolean isLiked;        // 현재 사용자 좋아요 여부
private String  memProfilePhoto;
private int     memRole;        // 회원 등급
```

`CommentsLikeController.java` — 댓글 좋아요

```jsx
// 게시글 좋아요와 동일하게 LOGIN_REQUIRED 문자열 응답
if (session == null || session.getAttribute("memNo") == null) {
    out.print("LOGIN_REQUIRED");
    return null;
}

int result = service.toggleCommentsLike(commentsId, memNo);  
int likeCount = service.getCommentsLikeCount(commentsId);    
out.print(likeCount);
```

`CommentsMapper.xml` — 댓글 수정 

```jsx
<update id="commentsUpdate">
    UPDATE COMMENTS
    SET COMMENTSCONTENT = #{commentsContent}
    WHERE COMMENTSID = #{commentsId}
      AND MEMNO = #{memNo}   ← SQL에서 직접 작성자 검증
</update>
```

`CommentsDeleteOkController.java`  — 댓글 트리 삭제 (2단계)

```jsx
// 1단계: 트리 전체의 COMMENTS_LIKE 먼저 삭제 (FK 제약 위반 방지)
CommentsServiceImpl.getInstance().deleteCommentLikesByCommentTree(cId);

// 2단계: 댓글 트리 전체 삭제 (부모 + 모든 자식)
int result = CommentsServiceImpl.getInstance().commentsDeleteTree(map);

-------------------------------------------------------------------------

// Mapper
// 1단계:
<delete id="deleteCommentLikesByCommentTree">
    DELETE FROM COMMENTS_LIKE
    WHERE COMMENTSID IN (
        SELECT COMMENTSID FROM COMMENTS
        START WITH COMMENTSID = #{value}
        CONNECT BY PRIOR COMMENTSID = PARENTBOARDID
    )
</delete>
// 삭제 대상 댓글 + 모든 하위 자식의 ID를 한 번에 수집

// 2단계:
<delete id="commentsDeleteTree">
    DELETE FROM COMMENTS
    WHERE COMMENTSID IN (
        SELECT COMMENTSID FROM COMMENTS
        START WITH COMMENTSID = #{commentsId}
        CONNECT BY PRIOR COMMENTSID = PARENTBOARDID
    )
    AND #{memNo} = (SELECT MEMNO FROM COMMENTS WHERE COMMENTSID = #{commentsId})
</delete>
// 삭제 권한: 최상위 댓글 작성자 본인만 가능, 별도 SELECT 없이 SQL 안에서 검증
```

