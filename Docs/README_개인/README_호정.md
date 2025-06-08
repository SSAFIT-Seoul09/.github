
# 최종 관통 PJT 고민점 및 느낀점

---

## ⚙백엔드

### Video 엔티티 관련

#### ✅ `from()` 메서드
- DTO → Entity 변환
- API 요청을 안전하게 DB에 저장 가능한 객체로 변환

#### ✅ `getPartValue()` 메서드
- enum 내부 value 값을 외부로 노출
- API 응답 또는 DB 저장용 value 추출

#### ✅ `TimeStamped` 상속
- 모든 엔티티에 생성일, 수정일 자동화
- JPA 생명주기 이벤트로 자동 갱신

---

### 로그 처리 - Slf4j + log.info()

#### ✅ 사용 목적
- 실행 흐름 추적
- 예외 디버깅
- 보안 이벤트 감시
- 성능 병목 분석

#### ✅ 로그 수준

| 수준 | 설명 |
|------|------|
| trace | 상세 디버깅 |
| debug | 개발 로그 |
| info | 사용자 흐름 기록 |
| warn | 경고 로그 |
| error | 예외 처리 로그 |

---

### 백엔드 예외 처리

#### 버튼으로 프론트에서 보내는 값들을 한정하는데도 세부적인 예외처리가 필요한 이유
- 클라이언트는 신뢰할 수 없음 (Postman 등)
- 추후 자유 입력, 모바일/외부 연동 대비
- 명확한 예외 클래스는 유지보수에 유리
- GlobalExceptionHandler 통해 JSON 에러 응답 제공

---

### ⚙ MyBatis - WHERE 1 = 1

#### ✅ 이유
- 조건 선택적 붙일 때 AND 시작 오류 방지
- 템플릿처럼 조건들을 자연스럽게 연결

```xml
WHERE 1 = 1
<if test="part != null"> AND part = #{part} </if>
```

---

### DB JOIN (리뷰 + 유저 + 비디오)

#### ✅ 문제
- 리뷰에 작성자 닉네임, 영상 제목 없음 → 프론트는 매번 추가 API 호출

#### ✅ 해결
- JOIN 쿼리로 `nickname`, `videoTitle` 포함
- DTO 필드 추가
- API 1회로 모든 데이터 수신 가능

---

### 댓글 기능 오류 및 수정

| 위치 | 문제 | 해결 |
|------|------|------|
| createComment | null 체크 반대로 작성 | `== null`로 수정 |
| findById | resultType 없음 | 추가 |
| updateComment | WHERE 누락 | `WHERE id = #{id}` 추가 |
| deleteById | 파라미터 이름 불일치 | `#{id}`로 수정 |

---

### 리뷰 삭제 오류

#### ✅ 원인
- 댓글이 존재해 리뷰 삭제 불가 (FK 제약)

#### ✅ 해결
1. 댓글 먼저 삭제 후 리뷰 삭제
2. DB에서 `ON DELETE CASCADE` 설정
3. 예외 처리 후 사용자 안내

---

### 🔍 비디오 상세페이지 오류

#### ✅ 문제
- 백엔드는 `id`, 프론트는 `videoId` 참조 → null 오류

#### ✅ 해결
- DTO에서 `v.id AS videoId` 설정
- 프론트에서 `video.id || video.videoId` 형태로 접근

---

## 🌐 프론트엔드 

### 주요 기능 구현

#### 유튜브 썸네일
- video URL에서 썸네일 추출 렌더링

#### 조회수 표시
- `views` 필드 사용
- VideoDetailView, VideoCardGrid에 뷰 표시
- localStorage 이용해 중복 조회 방지

```js
onMounted(async () => {
  const viewed = JSON.parse(localStorage.getItem("viewedVideos") || "[]");
  if (!viewed.includes(videoId)) {
    await videoStore.increaseViewCount(videoId);
    viewed.push(videoId);
    localStorage.setItem("viewedVideos", JSON.stringify(viewed));
  }
});
```

---

### 검색 및 필터링

#### ✅ 디바운스 검색
- lodash `debounce` 함수 사용 (300ms 지연)

#### ✅ 정렬 기능
- 최신순 / 조회순 선택 가능
- 선택된 순서에 따라 리스트 자동 정렬

---

### 마이페이지 영상 상세 이동 오류

#### ✅ 문제
- `video.videoId === null` → 상세페이지 이동 불가

#### ✅ 해결
- DTO에서 `v.id AS videoId` alias 명시
- 프론트는 기존 코드 유지 가능

```js
const navigateToVideo = (video) => {
  const videoId = video.videoId
  if (!videoId) {
    console.error('videoId 없음:', video)
    return
  }
  router.push(`/videos/${videoId}`)
}
```

---


### JWT 토큰 저장 및 인증 상태 유지 문제

#### ✅ 문제
- 백엔드(Spring Boot)에서 JWT를 쿠키에 담아 응답했지만, 프론트에서 이를 저장하거나 인증 상태로 인식하지 못함.

#### ✅ 해결
- 프론트엔드 Axios 요청 시 `withCredentials: true` 설정 추가
- 백엔드에서 CORS 설정 시 `allowCredentials(true)` 및 `allowedOriginPatterns("*")` 지정
- 사용자 정보를 로컬스토리지에 저장하는 방식 병행:
  ```js
  localStorage.setItem('access_token', token)
  ```

---

###  사용자 닉네임 한글 인코딩 깨짐

#### ✅ 문제
- JWT에서 nickname을 추출했을 때 한글이 깨진 채 표시됨 (`%ED%99%8D%EA%B8%B8%EB%8F%99` 형태)

#### ✅ 해결
- `decodeURIComponent()`를 통해 복호화 처리
  ```js
  const displayNickname = computed(() => {
    try {
      return decodeURIComponent(userNickname.value)
    } catch {
      return userNickname.value
    }
  })
  ```

---

### Axios 인터셉터 설정

#### ✅ 문제
- 모든 요청에 JWT를 자동으로 포함시키고 싶었으나, 일일이 헤더를 설정하는 번거로움 발생

#### ✅ 해결
- Axios 인터셉터를 활용해 자동으로 토큰 포함 설정
  ```js
  axios.interceptors.request.use(config => {
    const token = localStorage.getItem('access_token')
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  })
  ```

---

### 디바운스 적용 (검색창 최적화)

#### ✅ 문제
- 사용자가 검색창에 입력할 때마다 매번 API 요청 발생 → 과도한 네트워크 사용

#### ✅ 해결
- `debounce` 유틸함수 구현 후 `watch` 또는 `v-model` 이벤트에 적용
  ```js
  export const debounce = (func, wait) => {
    let timeout
    return (...args) => {
      clearTimeout(timeout)
      timeout = setTimeout(() => func(...args), wait)
    }
  }
  ```

---

### `v-for`에서 key 중복 오류

#### ✅ 문제
- 리스트 렌더링 시 콘솔에서 `Duplicate keys detected` 경고 발생

#### ✅ 해결
- 고유한 `id`를 `:key`로 지정 (예: `video.id`, `comment.id`)
  ```html
  <div v-for="item in items" :key="item.id"> ... </div>
  ```
---

### 전역 상태 관리

#### 1. 초기 설정 및 상태 관리 (favoriteStore.js)

```js
const favoriteVideos = ref(loadFavoritesFromStorage())
const isLoading = ref(false)
const error = ref(null)
const lastUpdate = ref(Date.now())

// localStorage에서 초기 상태 로드
const loadFavoritesFromStorage = () => {
  try {
    const stored = localStorage.getItem('favoriteVideos')
    return stored ? new Set(JSON.parse(stored).map(Number)) : new Set()
  } catch (err) {
    console.error('localStorage에서 찜 목록 로드 실패:', err)
    return new Set()
  }
}
```

- `favoriteVideos`: Set 자료구조로 찜한 비디오 ID 관리 (중복 방지)
- 앱 시작 시 localStorage에서 기존 찜 목록을 불러옴
- 실패 시 빈 Set으로 초기화

---

#### 2. 찜하기 토글 기능

```js
const toggleFavorite = async (videoId) => {
  if (!videoId) {
    console.error('videoId가 없습니다.')
    return false
  }
  const id = Number(videoId)

  // Optimistic Update (즉각적인 UI 반응)
  const isCurrentlyFavorite = favoriteVideos.value.has(id)
  if (isCurrentlyFavorite) {
    favoriteVideos.value.delete(id)
  } else {
    favoriteVideos.value.add(id)
  }
  saveFavoritesToStorage()

  try {
    const { data } = await axios.post(`/api/favorites/${id}`)
    if (data.success) {
      return !isCurrentlyFavorite
    } else {
      // API 실패 시 상태 롤백
      if (isCurrentlyFavorite) {
        favoriteVideos.value.add(id)
      } else {
        favoriteVideos.value.delete(id)
      }
      saveFavoritesToStorage()
      return isCurrentlyFavorite
    }
  } catch (err) {
    // 에러 처리 및 롤백
  }
}
```

- Optimistic Update 패턴 사용: 서버 응답 전에 UI 먼저 업데이트
- 실패 시 자동 롤백으로 데이터 일관성 유지
- localStorage 동기화로 브라우저 새로고침 후에도 상태 유지

---

#### 3. 전역 초기화 (App.vue)

```js
const initializeState = async () => {
  console.log('전역 상태 초기화 시작')
  await favoriteStore.initializeFavorites()
  videoStore.updateVideosFavoriteStatus()
  console.log('전역 상태 초기화 완료')
}

// 라우트 변경 시 찜하기 상태 유지
watch(
  () => route.path,
  async () => {
    console.log('라우트 변경 감지:', route.path)
    await initializeState()
  }
)

// 앱 마운트 시 초기 데이터 로드
onMounted(async () => {
  console.log('App 마운트: 찜하기 상태 초기화')
  await initializeState()
})
```

- 앱 시작 시, 라우트 변경 시 찜 상태 초기화
- 비디오 목록의 찜 상태도 함께 업데이트

---

#### 4. 비디오 목록과 찜하기 상태 연동 (videoStore.js)

```js
const fetchVideos = async () => {
  try {
    const response = await axios.get(`/api/videos/search?${queryParams.toString()}`)
    if (response.data.success) {
      const mappedVideos = response.data.data.map(video => {
        const videoId = video.id || video.videoId
        return {
          // ... 다른 비디오 정보
          isFavorite: favoriteStore.isFavorite(videoId)
        }
      })
    }
  } catch (err) {
    // 에러 처리
  }
}

// 찜 상태 동기화
const updateVideosFavoriteStatus = () => {
  videos.value = videos.value.map(video => ({
    ...video,
    isFavorite: favoriteStore.isFavorite(video.id || video.videoId)
  }))

  allVideos.value = allVideos.value.map(video => ({
    ...video,
    isFavorite: favoriteStore.isFavorite(video.id || video.videoId)
  }))
}
```

- 비디오 목록 조회 시 각 비디오의 찜 상태 자동 설정
- 찜 상태 변경 시 모든 비디오 목록의 상태 동기화

---

#### 5. 성능 최적화

```js
const initializeFavorites = async (force = false) => {
  if (!force && Date.now() - lastUpdate.value < 5000) {
    console.log('최근에 업데이트되어 초기화 스킵')
    return
  }

  isLoading.value = true
  try {
    const { data } = await axios.get('/api/favorites')
    if (data.success) {
      const favoriteIds = data.data.map(item => item.videoId)
      favoriteVideos.value = new Set(favoriteIds.map(Number))
      saveFavoritesToStorage()
    }
  } catch (err) {
    // 에러 처리
  }
}
```

- 불필요한 API 호출 방지 (5초 쿨다운)
- Set 자료구조로 빠른 찜 상태 확인
- localStorage 활용으로 서버 부하 감소
---

## 💬 느낀 점

### 백엔드 
JWT 토큰 기반 로그인, 팩토리 메소드, 예외처리 등을 고민해보면서 단순히 CRUD만을 구현하는 것이 아닌 백엔드단에서 어떻게 성능을 향상시키고,더 보기 좋은 코드를 제공할 수 있을지에 대해 팀원과 고민해보게 되었습니다.     
도메인 단위로 프로젝트 구조를 나누는 DDD(Domain-Driven Design)를 경험해보면서 도메인 단위로 협업하는 경험이 로직 구현이나 캡슐화에 대해 생각해볼 수 있는 기회라고 느꼈습니다.    
또, 단순히 기존에 사용했던 기술을 사용하는게 아니라 각각 어떤 장단점이 있을지 고민해보고 적용하는게 중요하다고 생각하게 되었습니다.    
예전에는 ERD나 API 명세서를 작성하는데 많은 시간을 할애하는 것보다는 빨리 작업을 시작하기에 급급했는데 기초를 잘 다지고 문서화를 잘 시켜놓으면 프론트-백 협업하기도 편하고 개발 시간 단축에 많은 도움이 된다고 느꼈습니다.    
마지막으로 API 공장이 되지 않기 위해 보안이나 에러 핸들링, 나아가서 부하 테스트나 배포/운영까지 공부해보면 많은 도움이 될 것 같다는 생각이 들었습니다.

### 프론트엔드
처음으로 프론트엔드 프레임워크를 사용해보았는데 초반에 컴포넌트를 잘 나누지 않아서 나중에 view가 많아지는 문제가 생긴 것 같아 약간의 아쉬움이 남았습니다.     
또 AI를 활용해서 페이지를 만들어가면서 AI의 발전이 정말 많이 이루어졌음을 깨달았습니다.    
UI와 프롬프트만 잘 작성하면 기초적인 부분에서는 많은 도움을 받을 수 있었습니다. 개인 성장에는 조금 걸림돌이 될 것 같지만 모르는 부분을 빠르게 해결해나가 프로젝트 완성도는 높아졌다고 생각합니다.       
이 과정 속에서 어떻게 하면 AI가 빠르게 발전하고 있는 이 상황에서 어떻게 나아가야할지도 고민해보게 되었습니다.     
아직까지는 볼륨이 커지고 세세한 프롬프트는 AI가 잘 인식하지 못하는 것 같아서 사람이 필수적으로 필요하다는 생각과 더불어     
개발을 해서 화면을 만들어내는 것도 중요하지만 사용자 경험에 대해 생각할 줄 아는 프론트엔드 개발자가 되어야 되겠다는 생각이 들었습니다.
마지막으로 백엔드 개발 뒤에 프론트엔드 개발이 남아있다는 것을 깨달으며 백엔드 개발자들이 빨리 API를 제공하면 참 좋겠다는 생각이 들었습니다.