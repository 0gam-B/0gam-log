# English patterns I keep relearning

A running list of English expressions that show up often in tech writing
but trip me up. I read this in the morning, a few minutes at a time.

---

## 1. `keep up` — 따라잡다, 보조 맞추다

"유지하다"가 아니라 **속도를 맞춰서 같이 가다**라는 뜻.

- *The consumer can't keep up.*
  컨슈머가 (속도를) 못 따라간다.
- *I can't keep up with you, you walk too fast.*
  너 너무 빨리 걸어서 못 따라가겠다.
- *Producers slow down when consumers can't keep up.*
  컨슈머가 못 따라오면 프로듀서가 느려진다.

**관련**: `keep up with the news` (뉴스를 따라잡다 / 흐름을 놓치지 않다)

---

## 2. `pair A with B` — A를 B와 짝지어라 / 같이 써라

명령형으로 자주 나옴. 기술 글에서 "X와 Y를 조합해서 쓰면 좋다"는 톤.

- *Pair it with a blocking wait strategy.*
  blocking wait strategy를 함께 써라.
- *Pair Kafka with a stream processor for real-time pipelines.*
  실시간 파이프라인엔 Kafka를 stream processor와 같이 써라.

**비슷한 표현**:
- `combine A with B` — 결합해라 (좀 더 중립적)
- `A works well with B` — A는 B와 잘 맞는다 (서술형)

---

## 3. `instead of A or B` — A하거나 B하는 대신

`instead of` 뒤에 동명사(-ing) 또는 명사가 옴.

- *Producers slow down instead of blocking on a lock.*
  프로듀서는 lock에 막히는 대신 (자연스럽게) 느려진다.
- *I write to local stage instead of HDFS directly.*
  HDFS에 직접 쓰는 대신 local stage에 쓴다.
- *Instead of either A or B, the system does C.*
  A나 B가 아니라 C를 한다. (둘 다 아니고 다른 옵션)

**주의**: `instead of` 다음에 동사 원형 안 옴. 동명사(-ing)가 와야 함.
- ❌ instead of block on a lock
- ✅ instead of blocking on a lock

---

## 4. `at <지점>` — ~지점에서 / ~할 때

`in`, `on`보다 "정확한 한 점"의 뉘앙스. 시간·위치·코드 지점 다 OK.

- *Producers slow down at `next()`.*
  `next()` 호출 지점에서 느려진다.
- *The error happens at line 42.*
  42번째 줄에서 에러가 난다.
- *We meet at 9 AM.*
  9시 정각에 만난다.
- *The bug shows up at startup.*
  시작 시점에 버그가 나타난다.

**구분**:
- `at` → 한 점 (시각, 정확한 위치)
- `in` → 범위 안 (`in March`, `in the file`)
- `on` → 표면·날짜 (`on Monday`, `on the page`)

---

## 5. `when <조건>` — ~할 때 / ~한 경우에

영어 글에서 조건·상황을 묘사할 때 자주. 한국어 "~하면"보다 약간 더 자연스러움.

- *When the consumer can't keep up, producers slow down.*
  컨슈머가 못 따라오면 프로듀서가 느려진다.
- *When HDFS is unavailable, uploads queue up locally.*
  HDFS가 안 되면 업로드는 로컬에 쌓인다.
- *Pair it with a blocking wait strategy when CPU usage matters.*
  CPU 사용량이 중요할 때는 blocking wait strategy를 같이 써라.

**비슷한 표현**:
- `if` → 가정 ("만약 ~라면")
- `when` → 시점·상황 ("~할 때")
- `whenever` → 그럴 때마다 (반복 강조)

`when`은 일이 실제 일어난다는 전제, `if`는 일어날지 모른다는 전제예요.
- *If HDFS goes down...* (다운될 수도 있다는 가정)
- *When HDFS goes down...* (다운될 거라는 전제)

---

## 6. (다음에 발견하면 추가)
