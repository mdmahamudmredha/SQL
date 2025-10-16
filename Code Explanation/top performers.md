```sql

/*
Question 3: Top Performers					
Find the top 5 students in each exam based on their actual mark.					
	- Only include rows where rank ≤ 5				
	- Return columns: exam_name, auth_user_id, actual_mark, rank				
*/

-- Step 1: Best attempt per user per exam
WITH best_attempts AS (
  SELECT
    es.user_id,
    e.exam_id,
    e.exam_name,
    (es.total_correct_answers * e.each_ques_mark) -
    (es.total_false_answers * e.per_ques_negative_marking) AS actual_mark,
    TIMESTAMPDIFF(SECOND, es.user_exam_starts_at, es.user_exam_ends_at) AS exam_duration_seconds,
    ROW_NUMBER() OVER (
      PARTITION BY es.user_id, e.exam_id
      ORDER BY
        (es.total_correct_answers * e.each_ques_mark) -
        (es.total_false_answers * e.per_ques_negative_marking) DESC,
        TIMESTAMPDIFF(SECOND, es.user_exam_starts_at, es.user_exam_ends_at) ASC
    ) AS rn
  FROM exam_sessions AS es
  JOIN exams AS e ON es.exam_id = e.exam_id
),

-- Step 2: Rank top 5 students per exam (using only their best attempt)
ranked_best AS (
  SELECT
    exam_name,
    exam_id,
    user_id AS auth_user_id,
    actual_mark,
    exam_duration_seconds,
    RANK() OVER (
      PARTITION BY exam_id
      ORDER BY actual_mark DESC, exam_duration_seconds ASC
    ) AS ranking
  FROM best_attempts
  WHERE rn = 1
)

-- Step 3: Final selection
SELECT
  exam_name,
  auth_user_id,
  actual_mark,
  exam_duration_seconds,
  ranking
FROM ranked_best
WHERE ranking <= 5
ORDER BY exam_name, ranking;






-- to understand that how rank function works
/*
-- Step 1: Best attempt per user per exam
WITH best_attempts AS (
  SELECT
    es.user_id,
    e.exam_id,
    e.exam_name,

    -- Actual mark
    (es.total_correct_answers * e.each_ques_mark)
      - (es.total_false_answers * e.per_ques_negative_marking) AS actual_mark,

    -- Duration in seconds
    TIMESTAMPDIFF(SECOND, es.user_exam_starts_at, es.user_exam_ends_at) AS exam_duration_seconds,

    -- Row number to get best attempt per user per exam
    ROW_NUMBER() OVER (
      PARTITION BY es.user_id, e.exam_id
      ORDER BY 
        (es.total_correct_answers * e.each_ques_mark)
          - (es.total_false_answers * e.per_ques_negative_marking) DESC,
        TIMESTAMPDIFF(SECOND, es.user_exam_starts_at, es.user_exam_ends_at) ASC
    ) AS rn
  FROM exam_sessions AS es
  JOIN exams AS e
    ON es.exam_id = e.exam_id
)

-- Step 2: Rank top students in each exam
SELECT
  exam_name,
  user_id AS auth_user_id,
  actual_mark,
  exam_duration_seconds,
  RANK() OVER (
    PARTITION BY exam_id
    ORDER BY actual_mark DESC, exam_duration_seconds ASC
  ) AS ranking
FROM best_attempts
WHERE rn = 1
ORDER BY exam_name, ranking;
*/

```
## 🔹 Step 1: Best attempt per user per exam

```sql
WITH best_attempts AS (
  SELECT
    es.user_id,
    e.exam_id,
    e.exam_name,

    -- Actual mark হিসাব করা
    (es.total_correct_answers * e.each_ques_mark) - 
    (es.total_false_answers * e.per_ques_negative_marking) AS actual_mark,

    -- পরীক্ষা কতক্ষণ দিয়েছে (duration)
    TIMESTAMP_DIFF(es.user_exam_ends_at, es.user_exam_starts_at, SECOND) AS exam_duration_seconds,

    -- প্রতিটি student-এর জন্য best attempt বের করা
    ROW_NUMBER() OVER (
      PARTITION BY es.user_id, e.exam_id
      ORDER BY 
        (es.total_correct_answers * e.each_ques_mark) - 
        (es.total_false_answers * e.per_ques_negative_marking) DESC, -- Highest mark আগে
        TIMESTAMP_DIFF(es.user_exam_ends_at, es.user_exam_starts_at, SECOND) ASC -- যদি mark same হয় তবে যিনি কম সময়ে শেষ করেছেন তিনি এগিয়ে থাকবেন
    ) AS rn

  FROM
    `...exam_sessions` AS es
  JOIN
    `...exams` AS e
    ON es.exam_id = e.exam_id
)
```

### 👉 কাজ কী?

* একজন student একই exam একাধিকবার দিতে পারে।
* এখানে প্রত্যেক attempt-এর **actual_mark** (marking - negative marking) বের করা হচ্ছে।
* তারপর **duration** বের করা হচ্ছে (কত সময় লেগেছে)।
* **ROW_NUMBER()** দিয়ে একই user-exam pair এর মধ্যে কে কোন attempt best করেছে সেটা বের করা হচ্ছে।

### 👉 কেন দরকার?

আমরা চাই প্রতিটি student-এর exam-এ শুধু **best attempt** ধরা হোক। না হলে কেউ যদি একই exam বারবার দিয়ে থাকে তবে ডুপ্লিকেট হয়ে যাবে।

---

## 🔹 Step 2: Rank top 5 students per exam

```sql
ranked_best AS (
  SELECT
    exam_name,
    user_id AS auth_user_id,
    actual_mark,
    exam_duration_seconds,
    RANK() OVER (
      PARTITION BY exam_id
      ORDER BY actual_mark DESC, exam_duration_seconds ASC
    ) AS rank
  FROM best_attempts
  WHERE rn = 1
)
```

### 👉 কাজ কী?

* এখন আমরা প্রতিটি exam অনুযায়ী (PARTITION BY exam_id) ranking করছি।
* Ranking এর ভিত্তি:

  1. **Highest actual_mark** (বেশি mark আগে)
  2. যদি mark same হয় তবে **কম সময়ে শেষ করা ছাত্র আগে**
* এখানে `RANK()` ব্যবহার করা হয়েছে (মানে যদি একই mark+duration থাকে তবে দুইজন একই rank পাবে, আর rank skip হবে)।

### 👉 কেন দরকার?

আমরা প্রতিটি exam-এ কে **Top performer** সেটা বের করতে চাই। এজন্য প্রত্যেক exam অনুযায়ী র‌্যাঙ্ক করতে হবে।

---

## 🔹 Step 3: Final Selection

```sql
SELECT
  exam_name,
  auth_user_id,
  actual_mark,
  exam_duration_seconds,
  rank
FROM
  ranked_best
WHERE
  rank <= 5
ORDER BY
  exam_name,
  rank;
```

### 👉 কাজ কী?

* এখানে ফাইনালি **শুধু টপ ৫** র‌্যাঙ্ক পর্যন্ত নেয়া হয়েছে।
* অর্থাৎ প্রতিটি exam-এর শীর্ষ ৫ জন student কে বের করছে।
* `ORDER BY exam_name, rank` → প্রতিটি exam অনুযায়ী সাজানো থাকবে এবং rank অনুযায়ী ক্রমানুসারে আসবে।

---

## 🔹 পুরো লজিক একসাথে

1. **best_attempts**: প্রতিটি student-এর প্রতিটি exam-এর মধ্যে শুধু **best attempt** রেখে দেয়া।
2. **ranked_best**: প্রতিটি exam-এ best attempts নিয়ে ranking করা।
3. **Final select**: প্রতিটি exam-এর **top 5 performers** কে select করা।

---

👉 একটা বাস্তব উদাহরণ ধরুন:

* Exam A-তে 10 জন student দিয়েছে।
* কিন্তু কেউ কেউ ২-৩ বার দিয়েছে।
* Step 1 → সবার মধ্যে থেকে শুধু best attempt বেছে নিল।
* Step 2 → এখন ওই best attempt গুলো rank করলো।
* Step 3 → অবশেষে শীর্ষ ৫ জনকে দেখালো।

---

আপনি চাইলে আমি একটা **ছোট sample dataset** বানিয়ে এই কোয়েরি চালালে কী আউটপুট আসবে তা টেবিল আকারে দেখাতে পারি। সেটা চাইবেন?
