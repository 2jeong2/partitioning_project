아래는 전체 문서를 **정돈된 띄어쓰기**, **명확한 섹션 구분**, **일관된 크기 및 스타일**로 개선한 **Markdown 코드**입니다. GitHub, Notion 등에서 예쁘게 렌더링될 수 있도록 구성되어 있습니다:

---

````markdown
# 🎧 Spotify Songs 데이터 파티셔닝 및 인덱싱 성능 최적화

---

## 📌 개요

Spotify 곡 데이터를 `track_genre` 기준으로 **유사한 장르 그룹별 리스트 파티셔닝(List Partitioning)** 하였으며,  
자주 사용하는 `track_genre + artists` 조합에 대해 **인덱스를 생성**하여 **검색 성능을 최적화**하였습니다.

---

## 🛠️ 테이블 생성 및 파티셔닝 전략

<details>
<summary>🎵 테이블 생성 및 파티셔닝 SQL 보기</summary>

```sql
CREATE TABLE spotify_songs_partitioned (
  id INT,
  track_id VARCHAR(22),
  artists VARCHAR(600),
  album_name VARCHAR(300),
  track_name VARCHAR(600),
  popularity INT,
  duration_ms INT,
  explicit TINYINT(1),
  danceability FLOAT,
  energy FLOAT,
  key_col INT,
  loudness FLOAT,
  mode_col INT,
  speechiness FLOAT,
  acousticness FLOAT,
  instrumentalness FLOAT,
  liveness FLOAT,
  valence FLOAT,
  tempo FLOAT,
  time_signature INT,
  track_genre VARCHAR(50) NOT NULL,
  PRIMARY KEY(id, track_genre)
)
PARTITION BY LIST COLUMNS(track_genre) (
  PARTITION p_pop           VALUES IN ('pop', 'power-pop', 'pop-film', 'synth-pop', 'party'),
  PARTITION p_rock          VALUES IN ('rock', 'alt-rock', 'hard-rock', 'punk', 'punk-rock', 'grunge', 'psych-rock', 'rock-n-roll', 'garage', 'indie', 'indie-pop', 'emo', 'guitar', 'rockabilly'),
  PARTITION p_metal         VALUES IN ('metal', 'heavy-metal', 'death-metal', 'black-metal', 'grindcore', 'metalcore', 'hardcore', 'hardstyle'),
  PARTITION p_electronic    VALUES IN ('edm', 'electro', 'electronic', 'techno', 'trance', 'house', 'deep-house', 'progressive-house', 'minimal-techno', 'chicago-house', 'detroit-techno', 'dubstep', 'idm', 'drum-and-bass', 'breakbeat', 'club'),
  PARTITION p_hiphop_rnb    VALUES IN ('hip-hop', 'r-n-b', 'rap', 'funk', 'soul'),
  PARTITION p_jazz_classical VALUES IN ('jazz', 'classical', 'piano', 'instrumental', 'opera'),
  PARTITION p_world         VALUES IN ('k-pop', 'j-pop', 'j-rock', 'mandopop', 'cantopop', 'french', 'german', 'turkish', 'iranian', 'swedish', 'malay', 'latin', 'latino', 'spanish', 'brazil', 'mpb', 'forro', 'samba', 'pagode', 'sertanejo', 'world-music', 'indian'),
  PARTITION p_country_folk  VALUES IN ('country', 'bluegrass', 'honky-tonk', 'folk', 'singer-songwriter', 'songwriter'),
  PARTITION p_reggae        VALUES IN ('reggae', 'reggaeton', 'ska', 'dub', 'dancehall'),
  PARTITION p_child_kids    VALUES IN ('children', 'kids', 'disney'),
  PARTITION p_ambient_chill VALUES IN ('ambient', 'study', 'sleep', 'chill', 'new-age'),
  PARTITION p_misc          VALUES IN ('anime', 'blues', 'gospel', 'comedy', 'happy', 'sad', 'romance', 'show-tunes', 'alternative', 'groove', 'goth', 'trip-hop', 'tango', 'acoustic', 'british')
);
````

</details>

---

## 🔍 인덱싱 최적화

자주 사용하는 `track_genre + artists` 조합에 대해 인덱스를 생성하여 **검색 성능을 향상**시켰습니다.

### 🧠 인덱스란?

* 📖 **정의**: 책의 목차처럼, 테이블에서 특정 데이터를 빠르게 찾기 위한 **검색 도우미**
  → 전화번호부의 A-Z 색인과 유사한 역할

### 📌 주요 특징

* ✅ 검색 속도 향상
* ✅ 중복 방지
* ✅ 정렬 최적화

---

## ⚖️ 파티셔닝 vs 인덱스

| 항목        | 🔎 인덱스                      | 📂 파티셔닝                                 |
| --------- | --------------------------- | --------------------------------------- |
| **목적**    | 특정 **데이터를 빠르게 찾기 위함**       | 특정 **범위만 접근하게끔 함**                      |
| **적용 단위** | **컬럼 단위**                   | **테이블 전체 구조**                           |
| **장점**    | 빠른 검색 최적화                   | 쿼리 범위 제한, 유지보수 효율                       |
| **사용 예시** | `WHERE explicit = 1` 빠르게 찾기 | `WHERE popularity < 40` → 해당 범위 파티션만 조회 |

👉 **요약**:

* 파티션은 "**어디를** 볼지" 정하고
* 인덱스는 "**어떻게 빠르게** 찾을지"를 정함
* 🧩 둘을 함께 쓰면 범위도 줄이고, 탐색도 빠름

---

## 🔧 인덱스 생성

<details>
<summary>인덱스 생성 SQL 보기</summary>

```sql
CREATE INDEX idx_genre_artist 
ON spotify_songs_partitioned (track_genre, artists);
```

</details>

---

## 🔎 쿼리 예시 및 실행 계획

<details>
<summary>예시 쿼리 & EXPLAIN 보기</summary>

```sql
-- 예시 쿼리
SELECT track_name, popularity  
FROM spotify_songs_partitioned 
WHERE track_genre = 'pop' AND artists = 'Arko' 
ORDER BY track_name DESC 
LIMIT 10;

-- 실행 계획
EXPLAIN SELECT track_name, popularity  
FROM spotify_songs_partitioned 
WHERE track_genre = 'pop' AND artists = 'Arko' 
ORDER BY track_name DESC 
LIMIT 10;
```

</details>

### ✅ 실행 계획 결과

* `p_pop` 파티션만 스캔
* 인덱스 `idx_genre_artist` 사용 확인됨

![실행계획](https://github.com/user-attachments/assets/094468c0-0b39-44ce-89e0-0111ee530c98)

---

## ⚡ 성능 비교

**파티셔닝 및 인덱싱 적용 전후 실행 속도 비교**

![성능비교](https://github.com/user-attachments/assets/aa84296a-aaab-40bd-8693-b0fab74696ea)

| 항목  | 적용 전        | 적용 후                         |
| --- | ----------- | ---------------------------- |
| 파티션 | ❌ 전체 테이블 조회 | ✅ 해당 파티션만 접근                 |
| 인덱스 | ❌ 없음        | ✅ `track_genre + artists` 사용 |
| 성능  | ⏳ 느림        | ⚡ 대폭 향상                      |

---

> ✅ **결론**: 파티셔닝으로 범위를 줄이고, 인덱스로 그 안을 빠르게 검색하여 Spotify 곡 검색 성능을 효과적으로 최적화하였습니다.

```

---

이 마크다운을 복사하여 `.md` 파일로 저장하거나, GitHub, Notion 등에 붙여넣으면 보기 좋게 출력됩니다. 원하시면 PDF로 출력하거나 `.md` 파일도 만들어드릴 수 있어요!
```
