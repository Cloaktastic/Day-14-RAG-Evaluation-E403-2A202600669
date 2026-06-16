# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Summary of results from Exercise 3.2:

**Overall pass rate:** 20.00% (4 / 20 passed)

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.66 | 0.00 | 1.00 | 0.33 |
| Relevance | 0.31 | 0.00 | 0.75 | 0.24 |
| Completeness | 0.60 | 0.07 | 1.00 | 0.35 |
| Overall Score | 0.53 | 0.06 | 0.92 | 0.23 |

**Score interpretation:**
- Number of metrics in Good (0.8–1.0): 0
- Number of metrics in Needs Work (0.6–0.8): 2 (Faithfulness: 0.66, Completeness: 0.60)
- Number of metrics in Significant Issues (<0.6): 2 (Relevance: 0.31, Overall Score: 0.53)

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 4 | 25.0% |
| irrelevant | 10 | 62.5% |
| incomplete | 0 | 0.0% |
| off_topic | 2 | 12.5% |
| refusal | 0 | 0.0% |

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

### Failure 1 (ID: H03)

**Question:** How can I apply for credit transfer for a course taken at a foreign university?

**Agent Answer:** Course grade must be B+ or higher to apply for credit transfer.

**Scores:** Faithfulness: 0.11 | Relevance: 0.00 | Completeness: 0.07 | Overall: 0.06

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The overall evaluation score is extremely low (0.06), and the case is flagged as a hallucination failure. |
| Why 1 | Why did this occur? | The agent's response contains incorrect facts (hallucinating "B+ or higher" grade requirement) and misses all procedural steps. |
| Why 2 | Why did the agent generate incorrect facts? | The agent function is simulated, but in a real RAG system, the retriever either failed to retrieve the correct policy chunk or the generator ignored the correct threshold ("C or equivalent or higher"). |
| Why 3 | Why did the evaluation metrics score it so poorly? | The low relevance score (0.00) is caused by a lack of lexical overlap with the question tokens, and the low completeness score (0.07) is due to omitting the transfer submission procedure and syllabus requirements. |
| Why 4 | What is the root cause? | The generator system prompt does not enforce strict fidelity to the retrieved context, allowing the model to hallucinate incorrect grade criteria. Additionally, the lexical overlap evaluator is too sensitive to phrasing differences. |

**Root cause (from `find_root_cause()`):**
> `Answer does not address the question — improve prompt clarity`

**Do you agree with the root cause suggestion? Why?**
> Partly agree. The algorithm flags relevance as the lowest score (0.00), hence suggesting prompt clarity. In reality, this is a severe hallucination/faithfulness error ("B+" vs "C") combined with extreme incompleteness. The root cause is a failure of the generator to stick to the context (hallucination) and lack of detailed instructions to cover all steps in the expected answer.

**Proposed fix (specific, actionable):**
> - Enhance the system prompt of the generator to explicitly forbid making up grades/thresholds, requiring a direct quote of the criteria.
> - Use a semantic relevance evaluator (e.g., embeddings) instead of exact word overlap to avoid artificial 0.00 relevance scores when the answer is short but related.

---

### Failure 2 (ID: H05)

**Question:** What is the rule for a student who takes an unauthorized leave of absence for more than one semester?

**Agent Answer:** Students taking unauthorized leave are expelled permanently from the university.

**Scores:** Faithfulness: 0.29 | Relevance: 0.18 | Completeness: 0.12 | Overall: 0.20

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The overall score is very low (0.20), classified as a hallucination failure. |
| Why 1 | Why did this occur? | The agent claims students are "expelled permanently," which contradicts the actual rule ("classified as withdrawn and must apply for readmission"). |
| Why 2 | Why did the agent hallucinate the permanent expulsion rule? | The generator made a wild assumption or relied on pre-trained bias about disciplinary actions rather than the retrieved policy chunk. |
| Why 3 | Why did the evaluator score it low? | Faithfulness is low (0.29) because "expelled permanently" is not supported by the context. Completeness is low (0.12) because it misses the readmission process. |
| Why 4 | What is the root cause? | Lack of a strict grounding guardrail in the generator's prompt and the lack of a secondary validation step (hallucination filter) to reject unsupported claims before returning them. |

**Root cause (from `find_root_cause()`):**
> `Answer is missing key information — increase context window or improve generation`

**Do you agree with the root cause suggestion? Why?**
> Agree. The minimum score is Completeness (0.12), triggering this suggestion. The generator completely missed the critical instruction that the student is "classified as withdrawn and must apply for readmission", substituting it with "expelled permanently". Improving generation quality and prompt constraints is the key fix.

**Proposed fix (specific, actionable):**
> - Add strict negative constraints in the prompt: "If the exact outcome/status is not in the context, do not assume or exaggerate disciplinary outcomes (e.g., do not say expelled unless explicitly stated)."
> - Implement a self-correction/refinement loop where the model verifies if the generated status matches the retrieved policy.

---

### Failure 3 (ID: A03)

**Question:** How is the admission test score calculated at Fulbright University Vietnam?

**Agent Answer:** Admission test scores at Fulbright University Vietnam are calculated based on personal essays.

**Scores:** Faithfulness: 0.00 | Relevance: 0.75 | Completeness: 0.13 | Overall: 0.29

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The overall score is 0.29, classified as a hallucination failure due to a Faithfulness score of 0.00. |
| Why 1 | Why did this occur? | The agent attempts to answer how Fulbright calculations work, despite the context saying it is out of scope. |
| Why 2 | Why did the agent attempt to answer it? | The agent did not recognize the prompt as adversarial/trap or did not implement strict out-of-scope refusal logic. |
| Why 3 | Why did the evaluator score Faithfulness as 0.00? | The retrieved context contains only general info about VinUniversity and does not support the claim about Fulbright's personal essays. |
| Why 4 | What is the root cause? | Absence of an intent routing/classification gateway to detect and block out-of-scope questions, and the generator's tendency to be overly helpful instead of refusing. |

**Root cause (from `find_root_cause()`):**
> `Context is missing or irrelevant — improve retrieval`

**Do you agree with the root cause suggestion? Why?**
> Agree. The Faithfulness score is 0.00 because the context does not contain any information about Fulbright. While the retriever successfully retrieved VinUniversity context, it was irrelevant to the query. The generator should have refused to answer based on this mismatch.

**Proposed fix (specific, actionable):**
> - Implement a query intent classification step before the retrieval phase to identify out-of-scope queries (e.g., questions about other universities) and reject them immediately.
> - Configure the generator to return a standard out-of-scope refusal response if the retrieved context does not contain direct answers to the query.

---

## 3. Failure Clustering

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|------------|---------------------|----------|
| 1 | Out-of-scope / Adversarial queries not refused | A01, A02, A03 | High |
| 2 | Generator hallucinations (wrong rules, grades, penalties) | H01, H02, H03, H04, H05 | High |
| 3 | Lexical overlap heuristic limits (false low relevance / completeness) | E02, E04, M03, M04, M05, M07 | Medium |
| 4 | Incomplete answers (missing key requirements / exceptions) | M01, M02 | Medium |

**If you could only fix one cluster, which one would you choose? Why?**
> I choose **Cluster 2 (Generator hallucinations)**. In the context of academic regulations and admissions, providing false information (such as stating a student is permanently expelled instead of withdrawn, or requiring a B+ instead of a C for credit transfer) is the most critical risk. Hallucinations destroy user trust and can lead to severe academic and administrative disputes. Fixing this cluster by enforcing strict prompt grounding and verification will directly improve the Faithfulness metric, which is the absolute baseline of safety for any business/academic assistant.

---

## 4. Improvement Log

Output of `generate_improvement_log()`:

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | irrelevant | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F002 | irrelevant | Answer does not address the question — improve prompt clarity | Enforce strict grounding in prompt (e.g., 'only use the context provided') | Open |
| F003 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt and routing mechanism to improve question relevancy | Open |
| F004 | off_topic | Answer is missing key information — increase context window or improve generation | Use query expansion or HyDE to retrieve more relevant documents | Open |
| F005 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F006 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F007 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F008 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F009 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F010 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F011 | hallucination | Answer does not address the question — improve prompt clarity | N/A | Open |
| F012 | irrelevant | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F013 | hallucination | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F014 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F015 | hallucination | Multiple issues detected — review full pipeline | N/A | Open |
| F016 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |

**Actionable improvement suggestions from analysis:**
1. Implement hallucination checker to filter unsupported claims.
2. Enforce strict grounding in prompt (e.g., 'only use the context provided').
3. Refine system prompt and routing mechanism to improve question relevancy.
4. Use query expansion or HyDE to retrieve more relevant documents.

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Question 1: When should you run `run_regression()` in a production system?**
> It should be automatically triggered in the CI/CD pipeline when:
> 1. Code changes are made to the retrieval or generation components.
> 2. The system prompt is modified.
> 3. New data/documents are added or updated in the knowledge base.
> 4. The LLM model version is updated.

**Question 2: Is the regression threshold of 0.05 appropriate for your domain?**
> No, it is too lenient. For academic regulations and admissions (involving tuition fees, graduation criteria, disciplinary actions), a regression of 0.05 is unsafe. We should enforce a stricter threshold: `0.02` for relevance and completeness, and a strict `0.00` (zero tolerance) for faithfulness regressions to prevent any new hallucinations.

**Question 3: When a regression is detected, should you block deployment or just alert?**
> **Block deployment**. Serving incorrect academic regulations can lead to serious compliance, legal, and trust issues. Although blocking might slow down release speeds, correctness is paramount in this domain.

**Question 4: Where should the eval pipeline run in the CI/CD flow?**
> ```
> Code change → [Run unit tests] → [Run benchmark eval (20 QA)] → [Verify regression & thresholds] → Deploy
>                  (Step 1)                   (Step 2)                       (Step 3)
> ```

---

## 6. Continuous Improvement Loop

**Based on today's lab, the next 3 actions to improve the agent:**

| Priority | Action | Metric to improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Implement a Query Intent Classifier to detect out-of-scope / adversarial queries before retrieval. | Relevance, Safety | Eliminates irrelevant responses and prompt injection exploits. |
| 2 | Refine Generator System Prompt to enforce strict grounding and structured outputs (e.g., listing exact credits, document lists, and dates). | Completeness, Faithfulness | Significantly reduces hallucinations and missing information. |
| 3 | Integrate a semantic reranker (like BGE-Reranker) to order retrieved context. | Context Precision | Puts the most relevant information at the top of the context, improving generation quality. |

**What failure cases will you add to the benchmark for the next sprint?**
> - Comparative queries about changes in academic policies across different catalog years (e.g., graduation requirements for cohort 2024 vs 2025).
> - Queries using local student slang or abbreviations (e.g., "course withdrawal", "under probation", "scholarship renewal") to test retrieval robustness.

---

## 7. Framework Reflection

**Framework used in lab:** Custom Heuristic Evaluator (RAGAS-inspired)

**If used in production, which framework would you choose and why?**
> I choose **DeepEval**.

| Criteria | Why DeepEval? |
|----------|---------------|
| Focus alignment | Excellent support for unit testing with pytest-like assertions, allowing quick testing during development. |
| CI/CD integration | Command-line interface integrates seamlessly with GitHub Actions, automatically outputting visual reports for pull requests. |
| Team workflow | Confident AI dashboard allows team members to visualize evaluation histories, track metrics over time, and debug failure logs easily. |
