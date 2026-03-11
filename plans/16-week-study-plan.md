# 16-Week Job Switch Preparation Plan

**Target:** SDE-2 at mid-size product companies (Razorpay, CRED, Groww, PhonePe, Swiggy, Slice)
**Timeline:** March 11 - June 30, 2026 (16 weeks, ~10-12 hrs/week = ~170 total hours)
**Split:** 40% DSA (~68 hrs) | 25% System Design (~42 hrs) | 20% Java/Spring/SQL/Patterns (~34 hrs) | 15% Behavioral (~25 hrs)

---

## Phase 1: Foundation Building (Weeks 1-4, Mar 11 - Apr 7)

**Goal:** Build DSA pattern recognition, start System Design foundations, draft all STAR stories, study Java Core.

### Week 1 (Mar 11-16): Arrays + Two Pointers + GitHub Cleanup

**DSA (5-6 hrs):**
- Learn the Two Pointers and basic array patterns from NeetCode Roadmap
- Problems (15 total, mix of Easy/Medium):
  - Two Sum, Best Time to Buy/Sell Stock, Contains Duplicate, Product of Array Except Self
  - Two Sum II (sorted), 3Sum, Container With Most Water, Trapping Rain Water
  - Valid Palindrome, Remove Duplicates from Sorted Array
  - Move Zeroes, Merge Sorted Array, Majority Element
  - Rotate Array, Next Permutation
- **Method:** 25 min attempt -> if stuck, read approach -> code it yourself -> note the pattern

**System Design (1.5 hrs):**
- Hello Interview: Read fundamentals module (scaling, horizontal vs vertical, load balancing)
- Watch: "System Design Fundamentals" video on Hello Interview

**Java/Spring/SQL (1.5 hrs):**
- Java Core: OOP deep-dive, Collections internals (HashMap, ArrayList) -- see `java/core-qa.md` Q1-Q11
- SQL: Joins fundamentals (INNER, LEFT, RIGHT) -- see `sql/sql-qa.md` Q1-Q3

**Behavioral (1 hr):**
- Draft STAR stories #1 and #2 (see `behavioral/star-stories.md`)

**One-time (2-3 hrs on weekend):**
- GitHub profile cleanup (pin 6 repos, archive forks, update bio)

### Week 2 (Mar 17-23): Sliding Window + Binary Search

**DSA (5-6 hrs):**
- Problems (15 total):
  - Sliding Window: Longest Substring Without Repeating Chars, Minimum Window Substring, Longest Repeating Character Replacement, Permutation in String, Fruit Into Baskets, Max Sum Subarray of Size K, Smallest Subarray with Given Sum
  - Binary Search: Binary Search, Search Insert Position, Find Min in Rotated Sorted Array, Search in Rotated Sorted Array, Koko Eating Bananas, Find Peak Element, Search a 2D Matrix, Median of Two Sorted Arrays

**System Design (1.5 hrs):**
- Caching strategies (Redis, CDN, cache invalidation)
- Database scaling (replication, sharding, partitioning)

**Java/Spring/SQL (1.5 hrs):**
- Java Core: String internals, Exception handling, Generics -- see `java/core-qa.md` Q12-Q23
- SQL: Indexing fundamentals -- see `sql/sql-qa.md` Q8-Q10

**Behavioral (1 hr):**
- Draft STAR stories #3 and #4

### Week 3 (Mar 24-30): Stack + Queue + HashMap

**DSA (5-6 hrs):**
- Problems (15 total):
  - Stack: Valid Parentheses, Min Stack, Evaluate Reverse Polish Notation, Daily Temperatures, Largest Rectangle in Histogram, Car Fleet, Generate Parentheses
  - HashMap: Group Anagrams, Top K Frequent Elements, Valid Anagram, Longest Consecutive Sequence, Subarray Sum Equals K, Encode and Decode Strings, Find All Anagrams, Isomorphic Strings

**System Design (1.5 hrs):**
- Hello Interview: Design URL Shortener (first full design)

**Java/Spring/SQL (1.5 hrs):**
- Java Core: Equals/HashCode, Immutability, Serialization -- see `java/core-qa.md` Q24-Q36
- SOLID Principles: S and O -- see `design-patterns/solid-principles.md`
- SQL: Query optimization basics -- see `sql/sql-qa.md` Q14-Q16

**Behavioral (1 hr):**
- Draft STAR stories #5 and #6

### Week 4 (Apr 1-7): Linked List + Week 1-3 Revision

**DSA (5-6 hrs):**
- Linked List (8 problems): Reverse Linked List, Merge Two Sorted Lists, Linked List Cycle, Remove Nth Node, Reorder List, Merge K Sorted Lists, LRU Cache, Add Two Numbers
- Revision: Re-solve 10 hardest problems from weeks 1-3

**System Design (1.5 hrs):**
- Hello Interview: Design Rate Limiter

**Java/Spring/SQL (1.5 hrs):**
- SOLID Principles: L, I, D -- see `design-patterns/solid-principles.md`
- Design Patterns: Singleton, Factory, Builder -- see `design-patterns/design-patterns.md`
- SQL: Transactions, ACID -- see `sql/sql-qa.md` Q17-Q20

**Behavioral (1 hr):**
- Draft STAR stories #7 and #8
- **Milestone:** All 8 STAR stories drafted. Read them out loud once.

**Resume update (30 min):**
- Add "Side Projects" section

**Phase 1 Checkpoint:** Solve ~60% Easy, ~30% Medium. 2 designs studied. All STAR stories drafted. Java Core Q&A covered.

---

## Phase 2: Intermediate + Start Applying (Weeks 5-8, Apr 8 - May 5)

**Goal:** Build Tree/Graph skills, deepen System Design, start applying, study Java Advanced + Spring Boot.

### Week 5 (Apr 8-13): Binary Trees + BFS/DFS

**DSA (5-6 hrs):**
- Problems (14): Invert Binary Tree, Maximum Depth, Same Tree, Subtree, LCA, Level Order, Validate BST, Kth Smallest, Construct from Preorder+Inorder, Right Side View, Count Good Nodes, Diameter, Balanced, Serialize/Deserialize

**System Design (1.5 hrs):**
- Hello Interview: Design Twitter/News Feed

**Java/Spring/SQL (1.5 hrs):**
- Java Advanced: Multithreading basics (Thread, Runnable, synchronized, volatile) -- see `java/advanced-qa.md` Q1-Q5
- Design Patterns: Strategy, State -- see `design-patterns/design-patterns.md`

**Behavioral (1 hr):**
- Practice delivering stories 1-4 out loud

**Job Applications:**
- Start applying via LinkedIn, Naukri (5-8 warm-up applications)

### Week 6 (Apr 14-20): Graphs (BFS/DFS/Topo Sort)

**DSA (5-6 hrs):**
- Problems (12): Number of Islands, Clone Graph, Pacific Atlantic, Course Schedule I & II, Connected Components, Graph Valid Tree, Rotting Oranges, Walls and Gates, Word Ladder, Surrounded Regions, Max Area of Island, Redundant Connection

**System Design (1.5 hrs):**
- Hello Interview: Design Chat/Messaging System

**Java/Spring/SQL (1.5 hrs):**
- Java Advanced: ExecutorService, CompletableFuture, Locks -- see `java/advanced-qa.md` Q6-Q11
- Spring Boot: IoC, DI, Bean lifecycle -- see `spring-boot/spring-boot-qa.md` Q1-Q5

**Behavioral (1 hr):**
- Practice stories 5-8 + "Why leaving?", "Why this company?", "Where in 3 years?"

### Week 7 (Apr 21-27): Heaps + Greedy

**DSA (5-6 hrs):**
- Problems (14): Kth Largest, Last Stone Weight, K Closest Points, Task Scheduler, Merge K Lists (redo), Find Median, Design Twitter, Kadane's, Jump Game I & II, Gas Station, Hand of Straights, Merge Intervals, Non-overlapping Intervals, Meeting Rooms II

**System Design (1.5 hrs):**
- Hello Interview: Design Notification System

**Java/Spring/SQL (1.5 hrs):**
- Java Advanced: Java Memory Model, GC, Java 8 Streams -- see `java/advanced-qa.md` Q12-Q23
- Spring Boot: Auto-configuration, Profiles -- see `spring-boot/spring-boot-qa.md` Q9-Q14

**Behavioral (1 hr):**
- Mock behavioral interview (pramp.com or friend)

### Week 8 (Apr 28 - May 5): Backtracking + Phase 2 Revision

**DSA (5-6 hrs):**
- Problems (8-10): Subsets, Combination Sum, Permutations, Word Search, Palindrome Partitioning, Letter Combinations, N-Queens, Subsets II
- Revision: Re-solve 10-12 hard problems from weeks 5-7

**System Design (1.5 hrs):**
- Hello Interview: Design File Storage System (S3/Dropbox)

**Java/Spring/SQL (1.5 hrs):**
- Spring Boot: Security (JWT, Filter Chain) -- see `spring-boot/spring-boot-qa.md` Q15-Q20
- Spring Data JPA: N+1 problem, Lazy vs Eager -- see `spring-boot/spring-data-jpa-qa.md` Q1-Q8
- SQL: Window Functions -- see `sql/sql-qa.md` Q21-Q23

**Behavioral (1 hr):**
- Practice "Tell me about yourself" (2-min and 5-min versions)

**Phase 2 Checkpoint:** Solve ~50-60% Medium. 6 designs done. Taking real interviews. Java Advanced + Spring Boot core covered.

---

## Phase 3: Advanced + Active Interviews (Weeks 9-12, May 6 - Jun 2)

**Goal:** Crack DP basics, deepen SD with domain expertise, interview actively. Master Spring Data JPA + SQL.

### Week 9 (May 6-11): DP Fundamentals (1D DP)

**DSA (5-6 hrs):**
- Problems (12): Climbing Stairs, House Robber I & II, LIS, Coin Change, Max Product Subarray, Word Break, Decode Ways, Partition Equal Subset Sum, Min Cost Climbing Stairs, Palindromic Substrings, Longest Palindromic Substring

**System Design (1.5 hrs):**
- Hello Interview: Design Payment System (**use PayU experience**)

**Java/Spring/SQL (1.5 hrs):**
- Spring Boot Advanced: @Async, AOP, Exception Handling -- see `spring-boot/spring-boot-qa.md` Q21-Q31
- Design Patterns: Observer, Template Method, Adapter, Decorator -- see `design-patterns/design-patterns.md`

**Behavioral (1 hr):**
- Mock: "Walk me through a system you designed end to end"

### Week 10 (May 12-18): DP Continued (2D DP)

**DSA (5-6 hrs):**
- Problems (10-12): Unique Paths, LCS, Edit Distance, 0/1 Knapsack, Target Sum, Interleaving String, Distinct Subsequences, Best Time Buy/Sell with Cooldown, Coin Change II, Regex Matching

**System Design (1.5 hrs):**
- Hello Interview: Design E-commerce Order System

**Java/Spring/SQL (1.5 hrs):**
- Spring Data JPA: Transactions, Propagation, Isolation, Locking -- see `spring-boot/spring-data-jpa-qa.md` Q9-Q15
- SQL Practice: Solve 5 practice problems from `sql/sql-qa.md`

**Behavioral (1 hr):**
- Mock interview (curveball questions)

### Week 11 (May 19-25): Intervals + Trie + Mixed

**DSA (5-6 hrs):**
- Intervals (4), Trie (3), Mixed random Mediums (5-6)

**System Design (1.5 hrs):**
- Hello Interview: Design Search Autocomplete

**Java/Spring/SQL (1.5 hrs):**
- Spring Data JPA: Performance, Pagination, Auditing -- see `spring-boot/spring-data-jpa-qa.md` Q16-Q23
- Design Patterns: Proxy, Facade, Command -- see `design-patterns/design-patterns.md`
- SQL Practice: 5 more practice problems

**Behavioral (1 hr):**
- Company-specific cultural research + tailored answers

### Week 12 (May 26 - Jun 2): Phase 3 Revision

**DSA (5-6 hrs):**
- Identify weakest 3 patterns, solve 5-6 NEW problems each
- Timed practice: 2 Mediums in 45 minutes

**System Design (1.5 hrs):**
- Hello Interview: Design Location-Based Service

**Java/Spring/SQL (1.5 hrs):**
- Java 11/17 features review -- see `java/advanced-qa.md` Q24-Q27
- Full Spring Boot revision -- review all Q&A files

**Behavioral (1 hr):**
- Full 30-min mock behavioral round

**Phase 3 Checkpoint:** Solve ~60-70% Medium. 10 designs mastered. Spring Boot + JPA deep knowledge. Active interviewing.

---

## Phase 4: Peak Performance (Weeks 13-16, Jun 3 - Jun 30)

**Goal:** Peak interview readiness. Target dream companies. Full revision of everything.

### Week 13-14 (Jun 3-16): Full Revision Sprint

**DSA:** 1 random Medium daily (timed 25 min) + re-solve 20 hardest
**System Design:** Redesign 2 systems per week from scratch (timed 40 min, no notes)
**Java/Spring/SQL:** Speed review -- go through all Q&A files, test yourself
**Behavioral:** Final mock + polish all stories
**Design Patterns:** Review SOLID + all 13 patterns -- test yourself on UML and code

### Week 15-16 (Jun 17-30): Interview Peak

**DSA:** Only timed practice (2 problems in 45 min) + company-specific problems
**System Design:** Mock design interviews with friends
**Java/Spring:** Review before each interview (30 min quick scan)
**Behavioral:** Pre-interview review of stories + company research

---

## Weekly Schedule Template

```
Monday-Friday (1.5 hrs/day):
  - 45 min: DSA (1 problem with pattern study)
  - 15 min: System Design reading (Hello Interview)
  - 15 min: Java/Spring/SQL/Design Patterns study (rotate daily)
  - 5 min: Mentally review one STAR story

Saturday (3-4 hrs):
  - 1.5 hrs: DSA deep-dive (2-3 problems)
  - 1 hr: System Design (full design practice)
  - 30 min: Java/Spring/SQL deep-dive
  - 30 min: Behavioral (write or practice story)

Sunday (3-4 hrs):
  - 1.5 hrs: DSA revision (re-solve week's hardest)
  - 30 min: System Design review
  - 30 min: Design Patterns / SOLID study
  - 30 min: Behavioral or SQL practice queries
  - 30 min: Job applications, profile updates
```

---

## Success Metrics

- **Week 4:** Solve 60% Easy, 30% Medium. 2 designs. All STAR stories drafted. Java Core done.
- **Week 8:** Solve 50-60% Medium. 6 designs. Taking interviews. Java Advanced + Spring Boot core done.
- **Week 12:** Solve 60-70% Medium. 10 designs. Spring Data JPA + SQL mastered. Improving interview pass rate.
- **Week 16:** Consistently solve Mediums in 25 min. Design any system in 40 min. Behavioral answers polished. Full-stack Java knowledge confident.

---

## Key Resources

| Area | Primary | Secondary |
|------|---------|-----------|
| DSA | NeetCode 150 (neetcode.io) | LeetCode Top 150, Striver A2Z |
| System Design | Hello Interview (subscription) | Alex Xu Vol 1, ByteByteGo YouTube |
| Java Core | `java/core-qa.md` in this repo | Baeldung.com |
| Java Advanced | `java/advanced-qa.md` in this repo | Java Concurrency in Practice (book) |
| Spring Boot | `spring-boot/spring-boot-qa.md` | Spring.io docs, Baeldung |
| Spring Data JPA | `spring-boot/spring-data-jpa-qa.md` | Vlad Mihalcea blog |
| SQL | `sql/sql-qa.md` in this repo | LeetCode SQL 50 |
| Design Patterns | `design-patterns/` in this repo | Head First Design Patterns |
| SOLID | `design-patterns/solid-principles.md` | Uncle Bob's articles |
| Behavioral | `behavioral/` in this repo | Pramp (free mock interviews) |
| Job Applications | LinkedIn, Naukri, company pages | AngelList/Wellfound |
