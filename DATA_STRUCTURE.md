# 버디버디 데이터 구조 (Realtime Database)

웹용 메신저 · 웹용 홈피 · 모바일용 메신저 · 모바일용 홈피가 **모두 같은 데이터베이스**를 씁니다.
네 곳 중 한 곳에서 저장 위치나 형식을 바꾸면 다른 곳에서 데이터가 안 보일 수 있으니, 바꿀 때는 이 문서도 같이 고쳐 주세요.

보안 규칙은 `database.rules.json` 에 있습니다. (Firebase 콘솔 → Realtime Database → 규칙 에 붙여 넣기)

## 로그인

- Firebase Authentication의 **이메일/비밀번호** 방식을 씁니다. 화면에서는 아이디만 입력합니다.
- 아이디 → 이메일 변환: `아이디(소문자)@buddybuddy.app`
  - 영문/숫자/`._-` 외의 글자가 있는 아이디는 `u-<UTF-8 16진수>@buddybuddy.app`
- 비밀번호 → Firebase 비밀번호 변환: `'bb#' + 비밀번호` (4자리 비밀번호도 쓸 수 있게)
- 모바일 앱도 **똑같은 변환**을 써야 같은 계정으로 로그인됩니다. (`idToEmail`, `toAuthPassword` 함수 참고)
- 비밀번호는 데이터베이스에 저장하지 않습니다.
- **이메일을 등록한 계정**은 Firebase 계정 이메일이 실제 이메일로 바뀝니다. 로그인할 때는
  `login_emails/{아이디 소문자}` 의 이메일 → 아이디 이메일 순서로 시도합니다.
- 비밀번호 재설정 메일(Firebase 화면)로 바꾼 비밀번호는 `bb#` 없이 저장되므로, 로그인할 때 그대로도 한 번
  시도하고 성공하면 `bb#` 형식으로 바꿔 둡니다.

## 계정 연결

```
id_owners/{아이디}: "<Firebase uid>"        // 이 아이디의 주인
owners_by_uid/{Firebase uid}: "{아이디}"     // 로그인한 사람의 아이디 찾기
login_emails/{아이디 소문자}: "실제 이메일"        // 비밀번호 찾기용. 아이디 하나씩은 누구나 조회 가능(목록은 불가), 본인만 수정
legacy_passwords/{아이디}: "옛 비밀번호"       // 옛 방식 계정만. 아무도 못 읽음. 첫 로그인 때 자동 삭제
migration_proofs/{아이디}: "..."             // 옛 계정을 옮길 때 잠깐 쓰고 바로 삭제
```

## 메신저

```
global_users/{아이디}
  ├ userId, nickname
  ├ status: "online" | "away" | "offline"
  └ profile: { gender, age, region, bio, avatar }   // avatar는 240px 이하 JPEG(data URL)

user_friends/{아이디}/{친구 아이디}: true            // 서로 수락하면 양쪽에 생김
friend_requests/{받는 사람}/{보낸 사람}: { fromId, fromNickname, timestamp }
rooms/{아이디A}_room_{아이디B}/messages/{자동키}: { senderId, senderNick, text }
      // 방 이름의 두 아이디는 알파벳 순서로 정렬
```

## 📮 문의함 (버디 우체통)

```
inquiries/{보낸 사람 아이디}/{자동키}
  ├ authorId, authorName
  ├ category: "question" | "bug" | "idea" | "etc"
  ├ title (50자 이하), content (2000자 이하), timestamp
  ├ status: "new" | "read" | "answered"
  ├ reply, replyAt            // 관리자만 쓸 수 있음
  └ userSeen                  // 보낸 사람이 답장을 확인했는지
```

- 관리자 아이디: `heartBROKENPlaYBoY` (보안 규칙과 메신저 코드의 `BUDDY_ADMIN_ID` 둘 다에 적혀 있음)
- 관리자: 모든 편지 읽기/답장/삭제. 보낸 사람: 자기 편지만 읽기/삭제, 답장 확인 표시만 가능

## 미니홈피

홈피 주소: `https://k2w0.github.io/buddybudddy-home/?uid={아이디}`

```
minihompy/{아이디}
  ├ data/{항목}: "JSON 문자열"        // 주인만 저장 가능. 주인과 서로 친구인 사람만 읽기 가능
  │    ├ profileData, mood
  │    ├ diaries, diaryCategories
  │    ├ boardPosts, boardCategories
  │    ├ jukeboxCategories, jukeboxSongs   // 내 컴퓨터 음악 파일(blob:)은 저장 안 됨
  │    ├ guestbookCategories
  │    ├ favoriteCategories, favoriteHomes
  │    ├ ilchonGroups   // [{ id, name, friends: [{ userId, name }] }]
  │    ├ photoAlbums    // [{ name, visibility, photos: [{ id, thumb, caption, date }] }]  thumb = 220px 작은 그림
  │    └ miniroom       // { customSkins: [{ name, url }], current }
  ├ photos/{사진 id}: "data URL"     // 1200px 이하 큰 그림. 크게 볼 때만 불러옴
  └ guestbook/{자동키}: { userId, userName, category, content, date, timestamp }
                                     // 친구만 자기 이름으로 쓰기 가능. 주인과 쓴 사람은 삭제 가능
```

### 공개 설정 (🔒 비공개 / 👥 친구공개 / 🌐 전체공개)

다이어리 · 게시글 · 사진첩(앨범)은 항목마다 `visibility` 를 갖고, **공개 범위에 따라 저장하는 칸이 다릅니다.**
화면에서 숨기는 게 아니라 칸이 달라서, 친구 브라우저에는 비공개 항목이 아예 내려가지 않습니다.

| visibility | 저장 칸 | 읽을 수 있는 사람 | 쓰는 항목 |
|---|---|---|---|
| `private`  | `hompy_private/{아이디}/data/{항목}` | 주인만 | diaries, boardPosts, photoAlbums |
| `friends`  | `minihompy/{아이디}/data/{항목}`     | 주인 + 서로 친구 | diaries, boardPosts, photoAlbums |
| `public`   | `hompy_open/{아이디}/data/boardPosts` | 로그인한 사용자 누구나 | boardPosts 만 |

```
hompy_private/{아이디}
  ├ data/{diaries | boardPosts | photoAlbums}: "JSON 문자열"   // 비공개 항목만
  └ photos/{사진 id}: "data URL"                              // 비공개 앨범의 큰 사진
hompy_open/{아이디}
  └ data/boardPosts: "JSON 문자열"                             // 전체공개 게시글만
```

- 각 칸의 배열 항목에는 원래 순서 `_seq` 가 붙어 있고, 주인 화면에서는 세 칸을 합쳐 `_seq` 순으로 정렬합니다.
- 공개 설정이 없는 옛 항목: 게시글/앨범은 친구공개, 다이어리는 주인이 접속할 때 비공개로 옮겨집니다.
- 새 다이어리 · 앨범 · 게시글의 기본값은 비공개입니다.
- 앨범 공개 설정을 바꾸면 큰 사진(`photos/{id}`)도 해당 칸으로 옮깁니다.
- 친구가 아닌 사람이 홈피에 오면 가림막 아래에 `hompy_open` 의 전체공개 글만 보여 줍니다.
- 탈퇴할 때 `minihompy`, `hompy_private`, `hompy_open` 의 내 칸을 모두 지웁니다.

## 자유게시판 (모든 친구의 홈피에 함께 보이는 공용 게시판)

```
public_board/{자동키}
  ├ authorId, authorName      // 쓴 사람 (본인 이름으로만 쓸 수 있음)
  ├ title (100자 이하), content (HTML, 표시할 때 위험한 코드 제거)
  ├ date: "YYYY-MM-DD HH:MM"   // 한국(기기) 시간
  ├ timestamp                  // 정렬용 숫자
  └ scrapCount                 // 누구나 1씩만 올릴 수 있음
```

- 각 홈피 게시판의 "자유게시판" 카테고리와 "전체보기"에 함께 보입니다.
- 쓴 사람만 수정/삭제할 수 있습니다. 다른 카테고리 글은 공개 설정에 따라 각자 홈피 칸에 저장됩니다. (아래 '공개 설정' 참고)

- `minihompy/{아이디}` 전체는 **주인과 서로 친구인 사람만** 읽을 수 있습니다. (`user_friends/{아이디}/{내 아이디}` 가 true)
- 화면에 사용자 글자(이름, 제목, 카테고리 등)를 넣을 때는 반드시 `escapeFavoriteHtml()`,
  `onclick="함수(...)"` 안에 넣을 때는 `jsArg()` 를 쓰세요. (다른 사람 화면에서 코드가 실행되는 것을 막기 위해)

`data` 아래 항목은 배열이 비어도 사라지지 않도록 **JSON 문자열**로 저장합니다. 읽을 때 `JSON.parse` 하세요.

## 무료 플랜(Spark)에서 주의할 점

- 저장 용량 1GB, **다운로드 월 10GB**, 동시 접속 100명
- 홈피를 열면 `data` 전체를 내려받습니다. 게시판에 사진을 많이 넣으면 그만큼 커집니다.
- 큰 사진은 `photos` 에 따로 두고 볼 때만 내려받습니다.
