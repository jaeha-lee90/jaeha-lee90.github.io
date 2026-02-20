---
title: C++ 코딩테스트 Day1 정리 (LeetCode 4문제) + 클래스(OOP) 입문
author: JaeHa
date: 2026-02-20 09:40:00 -0800
categories: [Blogging, CodingTest]
tags: [C++, LeetCode, HashMap, Array, OOP, Class, 코딩테스트]
pin: false
published: true
---

오늘은 코딩테스트 감각을 올리기 위해 **LeetCode Day1 기본 4문제**를 C++로 정리하고,
추가로 **클래스 객체지향(OOP) 기초**를 아주 쉬운 예제로 복습했다.

핵심 목표는 아래 2가지였다.
- 알고리즘: 자료구조 패턴(해시/배열) 빠르게 꺼내 쓰기
- C++ 문법: class, 생성자, getter/setter, 캡슐화 감각 익히기

---

## 1) LeetCode Day1 - 푼 문제

### 문제 목록
1. Two Sum (Easy)
2. Contains Duplicate (Easy)
3. Valid Anagram (Easy)
4. Top K Frequent Elements (Medium)

---

## 2) 문제별 핵심 아이디어 + 코드

### 2-1. Two Sum

**아이디어**
- `target - nums[i]`를 해시맵에서 찾는다.
- 값 → 인덱스를 저장하면서 한 번 순회하면 된다.

```cpp
#include <vector>
#include <unordered_map>
using namespace std;

class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> idx; // 값 -> 인덱스

        for (int i = 0; i < (int)nums.size(); i++) {
            int need = target - nums[i];
            if (idx.find(need) != idx.end()) {
                return {idx[need], i};
            }
            idx[nums[i]] = i;
        }

        return {};
    }
};
```

- 시간복잡도: `O(n)`
- 공간복잡도: `O(n)`

---

### 2-2. Contains Duplicate

**아이디어**
- `unordered_set`에 값을 넣으면서 이미 존재하면 `true`.

```cpp
#include <vector>
#include <unordered_set>
using namespace std;

class Solution {
public:
    bool containsDuplicate(vector<int>& nums) {
        unordered_set<int> seen;

        for (int x : nums) {
            if (seen.find(x) != seen.end()) return true;
            seen.insert(x);
        }

        return false;
    }
};
```

- 시간복잡도: `O(n)`
- 공간복잡도: `O(n)`

---

### 2-3. Valid Anagram

**아이디어**
- 길이가 다르면 바로 `false`.
- 알파벳 카운트 배열(26칸)로 `+1/-1` 후 0인지 확인.

```cpp
#include <string>
#include <vector>
using namespace std;

class Solution {
public:
    bool isAnagram(string s, string t) {
        if (s.size() != t.size()) return false;

        vector<int> cnt(26, 0);
        for (int i = 0; i < (int)s.size(); i++) {
            cnt[s[i] - 'a']++;
            cnt[t[i] - 'a']--;
        }

        for (int c : cnt) {
            if (c != 0) return false;
        }
        return true;
    }
};
```

- 시간복잡도: `O(n)`
- 공간복잡도: `O(1)` (고정 26)

---

### 2-4. Top K Frequent Elements

**아이디어**
- 1) 숫자 빈도 계산
- 2) 빈도를 인덱스로 쓰는 Bucket 생성
- 3) 높은 빈도부터 내려오며 `k`개 수집

```cpp
#include <vector>
#include <unordered_map>
using namespace std;

class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        unordered_map<int, int> freq;
        for (int x : nums) freq[x]++;

        int n = nums.size();
        vector<vector<int>> bucket(n + 1);

        for (auto& p : freq) {
            int num = p.first;
            int f = p.second;
            bucket[f].push_back(num);
        }

        vector<int> ans;
        for (int f = n; f >= 1 && (int)ans.size() < k; f--) {
            for (int num : bucket[f]) {
                ans.push_back(num);
                if ((int)ans.size() == k) break;
            }
        }
        return ans;
    }
};
```

- 시간복잡도: `O(n)`
- 공간복잡도: `O(n)`

---

## 3) C++ 클래스(OOP) 입문 - Student 예제

코테만 하다 보면 클래스 문법이 약해지기 쉬워서,
아주 쉬운 `Student` 예제로 **캡슐화 + 생성자 + getter/setter**를 연습했다.

```cpp
#include <iostream>
#include <string>
using namespace std;

class Student {
private:
    string name;
    int score;

public:
    // 생성자: 점수 범위 검증까지 같이 수행
    Student(string n, int s) {
        name = n;
        score = (0 <= s && s <= 100) ? s : 0;
    }

    // setter: 유효한 점수만 반영
    void setScore(int s) {
        if (0 <= s && s <= 100) {
            score = s;
        }
    }

    // getter: 읽기 전용 접근
    int getScore() const {
        return score;
    }

    string getName() const {
        return name;
    }

    void printInfo() const {
        cout << "Name: " << name << ", Score: " << score << '\n';
    }
};

int main() {
    Student st("Jaeha", 85);
    st.printInfo();

    st.setScore(95);
    st.printInfo();

    st.setScore(120); // 무시
    st.printInfo();
}
```

### 이 예제로 익힌 포인트
- 멤버 변수는 `private`로 숨기기 (캡슐화)
- 생성자에서 초기 상태를 안전하게 만들기
- setter에서 유효성 검증하기
- `const` 멤버함수로 읽기 전용 보장하기

---

## 4) 오늘 회고

- `unordered_map / unordered_set`를 쓰는 패턴은 확실히 손에 익혀야 한다.
- Easy라도 **직접 타이핑**으로 풀어야 실수가 줄어든다.
- OOP는 거창하게 시작하기보다, 작은 클래스 하나를 완성하는 방식이 훨씬 학습 효율이 좋다.

다음 목표는:
- LeetCode Day2 (Two Pointers / String)
- 클래스 심화: 생성자 초기화 리스트, 참조, 소멸자 기초

---

필요하면 다음 글에서 Day2도 같은 포맷으로 이어서 정리해보겠다.
