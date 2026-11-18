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

## 계정 연결

```
id_owners/{아이디}: "<Firebase uid>"        // 이 아이디의 주인
owners_by_uid/{Firebase uid}: "{아이디}"     // 로그인한 사람의 아이디 찾기
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

## 미니홈피

홈피 주소: `https://k2w0.github.io/buddybudddy-home/?uid={아이디}`

```
minihompy/{아이디}
  ├ data/{항목}: "JSON 문자열"        // 주인만 저장 가능. 로그인한 사람은 누구나 읽기 가능
  │    ├ profileData, mood
  │    ├ diaries, diaryCategories
  │    ├ boardPosts, boardCategories
  │    ├ jukeboxCategories, jukeboxSongs   // 내 컴퓨터 음악 파일(blob:)은 저장 안 됨
  │    ├ guestbookCategories
  │    ├ favoriteCategories, favoriteHomes
  │    ├ ilchonGroups   // [{ id, name, friends: [{ userId, name }] }]
  │    ├ photoAlbums    // [{ name, photos: [{ id, thumb, caption, date }] }]  thumb = 220px 작은 그림
  │    └ miniroom       // { customSkins: [{ name, url }], current }
  ├ photos/{사진 id}: "data URL"     // 1200px 이하 큰 그림. 크게 볼 때만 불러옴
  └ guestbook/{자동키}: { userId, userName, category, content, date, timestamp }
                                     // 로그인한 사람은 누구나 자기 이름으로 쓰기 가능
```

`data` 아래 항목은 배열이 비어도 사라지지 않도록 **JSON 문자열**로 저장합니다. 읽을 때 `JSON.parse` 하세요.

## 무료 플랜(Spark)에서 주의할 점

- 저장 용량 1GB, **다운로드 월 10GB**, 동시 접속 100명
- 홈피를 열면 `data` 전체를 내려받습니다. 게시판에 사진을 많이 넣으면 그만큼 커집니다.
- 큰 사진은 `photos` 에 따로 두고 볼 때만 내려받습니다.
