# RAG Pipeline Evaluation Results

Generated: 2026-09-01T04:22:57  
Model: `Qwen/Qwen2.5-3B-Instruct` | Embeddings: `all-MiniLM-L6-v2` | Device: `cuda`

## Headline results

| Metric                                   | Digital   | Scanned   | Overall   | Based on            |
|:-----------------------------------------|:----------|:----------|:----------|:--------------------|
| Answer accuracy                          | 77%       | 85%       | 81%       | 26 graded runs      |
| Correct refusal on trick questions       | 100%      | 100%      | 100%      | 8 control runs      |
| Citation accuracy (right source cited)   | 100%      | 100%      | 100%      | 20 cited answers    |
| Citation validity (source was retrieved) | 100%      | 100%      | 100%      | 22 cited answers    |
| Factual consistency (self-judged)        | 65%       | 65%       | 65%       | 34 judged answers   |
| Retrieval Recall@3                       | 93%       | 89%       | 91%       | 12 graded questions |
| Retrieval Hit Rate@3                     | 100%      | 100%      | 100%      | 12 graded questions |
| Mean Reciprocal Rank (MRR)               | 0.94      | 0.94      | 0.94      | 12 graded questions |
| Document-type classification recall      | 100%      | 100%      | 100%      | 4 files             |
| Avg response time                        | 4.4 sec   | 4.9 sec   | 4.7 sec   | 34 question runs    |
| Avg retrieval latency                    | 1003 ms   | 1010 ms   | 1006 ms   | 34 question runs    |
| Avg LLM generation time                  | 3.4 sec   | 3.9 sec   | 3.7 sec   | 34 question runs    |

## System architecture

`Document Input  ->  OCR  ->  Chunking  ->  Embeddings  ->  Vector Storage  ->  Retrieval  ->  LLM Response`

| Component       | Technology Choice                                                               | Configuration Details                                                                                                                                                                                                         |
|:----------------|:--------------------------------------------------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| OCR Engine      | Tesseract 5 via pytesseract, PyMuPDF for rasterisation                          | Triggered per page when the text layer has <40 characters; pages rendered at 200 DPI; English; digital text layer preferred when available                                                                                    |
| Text Chunking   | Custom sliding window (LlamaIndex SentenceSplitter available as an alternative) | 220-word chunks, 60-word overlap; chunks never span two sub-documents; each chunk carries doc type, exact page range, source file and digital/OCR flag                                                                        |
| Embeddings      | sentence-transformers/all-MiniLM-L6-v2                                          | 384-dimensional vectors, ~22M parameters, L2-normalised so inner product equals cosine similarity; runs on cuda                                                                                                               |
| Vector Database | FAISS (in-memory, IndexFlatIP)                                                  | Cosine similarity; exhaustive flat search (exact, no approximation); one global index plus one index per document type to support filtering                                                                                   |
| Retriever       | Dense vector search with LLM query routing                                      | top-k = 4 by default (1-10 in the UI); routing to a single document type when the LLM's confidence exceeds 0.7, otherwise global search; no reranking stage                                                                   |
| LLM             | Qwen/Qwen2.5-3B-Instruct (local, Hugging Face Transformers)                     | ~3.1B parameters, float16 on cuda; greedy decoding (temperature 0) for reproducibility; max 400 new tokens for answers, 20-60 for classification and routing                                                                  |
| Prompt Strategy | Zero-shot instruction prompts with labelled context injection                   | Retrieved passages are prefixed with [From DocType, Pages X-Y] labels the model is told to copy verbatim, which makes citations machine-checkable; an explicit refusal instruction covers questions the context cannot answer |

## Test set

| File                            | Variant   |   Pages |   Size (KB) |   Text-layer chars | Has text layer    |   Expected doc types |
|:--------------------------------|:----------|--------:|------------:|-------------------:|:------------------|---------------------:|
| pharma-blob-sample.pdf          | Digital   |      10 |        12.5 |               9687 | yes               |                    7 |
| sample-sdf-document.pdf         | Digital   |       3 |       143.7 |               4019 | yes               |                    2 |
| pharma-blob-sample_scanned.pdf  | Scanned   |      10 |     21264.8 |                  0 | no (OCR required) |                    7 |
| sample-sdf-document_scanned.pdf | Scanned   |       3 |      6167.7 |                  0 | no (OCR required) |                    2 |

## Document processing

| File                            | Variant   | Status   |   Pages |   OCR pages |   Docs found |   Chunks |   Types found |   Types expected |   Type recall | Missed types   | Spurious types   |   Wall time (s) |   OCR time (s) |
|:--------------------------------|:----------|:---------|--------:|------------:|-------------:|---------:|--------------:|-----------------:|--------------:|:---------------|:-----------------|----------------:|---------------:|
| pharma-blob-sample.pdf          | Digital   | OK       |      10 |           0 |           10 |       10 |             7 |                7 |             1 | -              | -                |             7.9 |            0   |
| sample-sdf-document.pdf         | Digital   | OK       |       3 |           0 |            3 |        3 |             2 |                2 |             1 | -              | -                |             1.8 |            0   |
| pharma-blob-sample_scanned.pdf  | Scanned   | OK       |      10 |          10 |           10 |       10 |             7 |                7 |             1 | -              | -                |            54.2 |           47.7 |
| sample-sdf-document_scanned.pdf | Scanned   | OK       |       3 |           3 |            2 |        5 |             2 |                2 |             1 | -              | -                |            19.2 |           17.7 |

## Retrieval performance

|   k | Variant   |   Queries |   Hit Rate |   Precision@K |   Recall@K |   MRR |
|----:|:----------|----------:|-----------:|--------------:|-----------:|------:|
|   1 | Digital   |        12 |      0.917 |         0.917 |      0.917 | 0.917 |
|   1 | Scanned   |        12 |      0.917 |         0.917 |      0.917 | 0.917 |
|   3 | Digital   |        12 |      1     |         0.5   |      0.931 | 0.944 |
|   3 | Scanned   |        12 |      1     |         0.556 |      0.889 | 0.944 |
|   5 | Digital   |        12 |      1     |         0.3   |      0.917 | 0.944 |
|   5 | Scanned   |        12 |      1     |         0.35  |      0.917 | 0.944 |

### Per-question detail (k=3)

| ID   | Variant   | Category       | File                            |   k | Hit   |   Precision@K |   Recall@K |    RR |   Relevant found |   Relevant in file | Expected type(s)                                | Retrieved type(s)                                                      | Routed to               |   Routing conf. | Scope                                       |
|:-----|:----------|:---------------|:--------------------------------|----:|:------|--------------:|-----------:|------:|-----------------:|-------------------:|:------------------------------------------------|:-----------------------------------------------------------------------|:------------------------|----------------:|:--------------------------------------------|
| Q01  | Digital   | factual_lookup | pharma-blob-sample.pdf          |   3 | True  |         0.333 |      1     | 1     |                1 |                  1 | Cover Letter                                    | Cover Letter                                                           | Cover Letter            |            0.85 | routed:Cover Letter                         |
| Q01  | Scanned   | factual_lookup | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.333 |      1     | 1     |                1 |                  1 | Cover Letter                                    | Cover Letter                                                           | Cover Letter            |            0.85 | routed:Cover Letter                         |
| Q02  | Digital   | factual_lookup | pharma-blob-sample.pdf          |   3 | True  |         0.667 |      1     | 1     |                2 |                  2 | Certificate Of Quality                          | Certificate Of Quality, Certificate Of Quality                         | Certificate Of Quality  |            0.9  | routed:Certificate Of Quality               |
| Q02  | Scanned   | factual_lookup | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.667 |      1     | 1     |                2 |                  2 | Certificate Of Quality                          | Certificate Of Quality, Certificate Of Quality                         | Certificate Of Quality  |            0.9  | routed:Certificate Of Quality               |
| Q03  | Digital   | factual_lookup | pharma-blob-sample.pdf          |   3 | True  |         0.667 |      1     | 1     |                2 |                  2 | Packaging Specification                         | Packaging Specification, Packaging Specification                       | Packaging Specification |            0.9  | routed:Packaging Specification              |
| Q03  | Scanned   | factual_lookup | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.667 |      1     | 1     |                2 |                  2 | Packaging Specification                         | Packaging Specification, Packaging Specification                       | Packaging Specification |            0.9  | routed:Packaging Specification              |
| Q04  | Digital   | compliance     | pharma-blob-sample.pdf          |   3 | True  |         0.333 |      1     | 1     |                1 |                  1 | Bse/Tse Declaration                             | Bse/Tse Declaration                                                    | Bse/Tse Declaration     |            0.9  | routed:Bse/Tse Declaration                  |
| Q04  | Scanned   | compliance     | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.333 |      1     | 1     |                1 |                  1 | Bse/Tse Declaration                             | Bse/Tse Declaration                                                    | Bse/Tse Declaration     |            0.9  | routed:Bse/Tse Declaration                  |
| Q05  | Digital   | factual_lookup | pharma-blob-sample.pdf          |   3 | True  |         0.333 |      1     | 1     |                1 |                  1 | Material Description                            | Material Description                                                   | Material Description    |            0.85 | routed:Material Description                 |
| Q05  | Scanned   | factual_lookup | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.333 |      1     | 1     |                1 |                  1 | Material Description                            | Material Description                                                   | Material Description    |            0.85 | routed:Material Description                 |
| Q06  | Digital   | compliance     | pharma-blob-sample.pdf          |   3 | True  |         0.667 |      1     | 1     |                2 |                  2 | Supplier Qualification                          | Supplier Qualification, Supplier Qualification                         | Supplier Qualification  |            0.9  | routed:Supplier Qualification               |
| Q06  | Scanned   | compliance     | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.667 |      1     | 1     |                2 |                  2 | Supplier Qualification                          | Supplier Qualification, Supplier Qualification                         | Supplier Qualification  |            0.9  | routed:Supplier Qualification               |
| Q07  | Digital   | factual_lookup | pharma-blob-sample.pdf          |   3 | True  |         0.333 |      1     | 1     |                1 |                  1 | Chain Of Custody                                | Chain Of Custody                                                       | Chain Of Custody        |            0.85 | routed:Chain Of Custody                     |
| Q07  | Scanned   | factual_lookup | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.333 |      1     | 1     |                1 |                  1 | Chain Of Custody                                | Chain Of Custody                                                       | Chain Of Custody        |            0.85 | routed:Chain Of Custody                     |
| Q08  | Digital   | multi_hop      | pharma-blob-sample.pdf          |   3 | True  |         0.333 |      0.5   | 1     |                1 |                  2 | Material Description, Bse/Tse Declaration       | Material Description                                                   | Material Description    |            0.85 | routed:Material Description                 |
| Q08  | Scanned   | multi_hop      | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.333 |      0.5   | 1     |                1 |                  2 | Material Description, Bse/Tse Declaration       | Material Description                                                   | Material Description    |            0.85 | routed:Material Description                 |
| Q09  | Digital   | multi_hop      | pharma-blob-sample.pdf          |   3 | True  |         0.667 |      0.667 | 1     |                2 |                  4 | Packaging Specification, Supplier Qualification | Supplier Qualification, Supplier Qualification                         | Supplier Qualification  |            0.85 | routed:Supplier Qualification               |
| Q09  | Scanned   | multi_hop      | pharma-blob-sample_scanned.pdf  |   3 | True  |         0.667 |      0.667 | 1     |                2 |                  4 | Packaging Specification, Supplier Qualification | Supplier Qualification, Supplier Qualification                         | Supplier Qualification  |            0.85 | routed:Supplier Qualification               |
| Q15  | Digital   | factual_lookup | sample-sdf-document.pdf         |   3 | True  |         0.667 |      1     | 1     |                2 |                  2 | Certificate Of Quality                          | Certificate Of Quality, Certificate Of Quality                         | Certificate Of Quality  |            0.85 | routed:Certificate Of Quality               |
| Q15  | Scanned   | factual_lookup | sample-sdf-document_scanned.pdf |   3 | True  |         1     |      1     | 1     |                3 |                  3 | Certificate Of Quality                          | Certificate Of Quality, Certificate Of Quality, Certificate Of Quality | Certificate Of Quality  |            0.85 | routed:Certificate Of Quality               |
| Q16  | Digital   | factual_lookup | sample-sdf-document.pdf         |   3 | True  |         0.333 |      1     | 0.333 |                1 |                  1 | Cover Letter                                    | Certificate Of Quality, Certificate Of Quality, Cover Letter           | Other                   |            0.85 | global (routing confidence below threshold) |
| Q16  | Scanned   | factual_lookup | sample-sdf-document_scanned.pdf |   3 | True  |         0.333 |      0.5   | 0.333 |                1 |                  2 | Cover Letter                                    | Certificate Of Quality, Certificate Of Quality, Cover Letter           | Other                   |            0.85 | global (routing confidence below threshold) |
| Q17  | Digital   | multi_hop      | sample-sdf-document.pdf         |   3 | True  |         0.667 |      1     | 1     |                2 |                  2 | Certificate Of Quality                          | Certificate Of Quality, Certificate Of Quality                         | Certificate Of Quality  |            0.95 | routed:Certificate Of Quality               |
| Q17  | Scanned   | multi_hop      | sample-sdf-document_scanned.pdf |   3 | True  |         1     |      1     | 1     |                3 |                  3 | Certificate Of Quality                          | Certificate Of Quality, Certificate Of Quality, Certificate Of Quality | Certificate Of Quality  |            0.95 | routed:Certificate Of Quality               |

## End-to-end results

| ID   | Variant   | Category             | Answerable   | Answer correct   | Grading mode   |   Keyword coverage | Has citation   | Citations valid   | Citations correct   | Cited                                    | Retrieved                                                        | Faithfulness   |   Mean similarity |   Chunks used |   Retrieval (s) |   Generation (s) |   Response time (s) |
|:-----|:----------|:---------------------|:-------------|:-----------------|:---------------|-------------------:|:---------------|:------------------|:--------------------|:-----------------------------------------|:-----------------------------------------------------------------|:---------------|------------------:|--------------:|----------------:|-----------------:|--------------------:|
| Q01  | Digital   | factual_lookup       | True         | True             | keyword        |               0.67 | True           | True              | True                | cover letter                             | Cover Letter                                                     | SUPPORTED      |             0.827 |             1 |           0.941 |            1.611 |                2.55 |
| Q01  | Scanned   | factual_lookup       | True         | True             | keyword        |               0.67 | True           | True              | True                | cover letter                             | Cover Letter                                                     | SUPPORTED      |             0.827 |             1 |           0.902 |            2.02  |                2.92 |
| Q02  | Digital   | factual_lookup       | True         | True             | keyword        |               0.67 | True           | True              | True                | certificate of quality                   | Certificate Of Quality                                           | SUPPORTED      |             0.68  |             2 |           0.907 |            2.652 |                3.56 |
| Q02  | Scanned   | factual_lookup       | True         | True             | keyword        |               0.67 | True           | True              | True                | certificate of quality                   | Certificate Of Quality                                           | SUPPORTED      |             0.68  |             2 |           0.891 |            4.193 |                5.08 |
| Q03  | Digital   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | packaging specification                  | Packaging Specification                                          | SUPPORTED      |             0.479 |             2 |           0.935 |            1.669 |                2.6  |
| Q03  | Scanned   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | packaging specification                  | Packaging Specification                                          | SUPPORTED      |             0.492 |             2 |           0.911 |            1.672 |                2.58 |
| Q04  | Digital   | compliance           | True         | False            | keyword        |               0    | False          |                   |                     | -                                        | Bse/Tse Declaration                                              | UNSUPPORTED    |             0.487 |             1 |           0.984 |            0.904 |                1.89 |
| Q04  | Scanned   | compliance           | True         | False            | keyword        |               0    | False          |                   |                     | -                                        | Bse/Tse Declaration                                              | UNSUPPORTED    |             0.487 |             1 |           1.323 |            0.914 |                2.24 |
| Q05  | Digital   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | material description                     | Material Description                                             | SUPPORTED      |             0.663 |             1 |           0.915 |            1.969 |                2.88 |
| Q05  | Scanned   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | material description                     | Material Description                                             | SUPPORTED      |             0.667 |             1 |           0.917 |            3.479 |                4.4  |
| Q06  | Digital   | compliance           | True         | True             | keyword        |               1    | True           | True              | True                | supplier qualification                   | Supplier Qualification                                           | SUPPORTED      |             0.598 |             2 |           0.922 |            5.328 |                6.25 |
| Q06  | Scanned   | compliance           | True         | True             | keyword        |               1    | True           | True              | True                | supplier qualification                   | Supplier Qualification                                           | SUPPORTED      |             0.598 |             2 |           0.926 |            5.425 |                6.35 |
| Q07  | Digital   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | chain of custody                         | Chain Of Custody                                                 | SUPPORTED      |             0.49  |             1 |           0.994 |            6.82  |                7.81 |
| Q07  | Scanned   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | chain of custody                         | Chain Of Custody                                                 | SUPPORTED      |             0.49  |             1 |           1.011 |            6.547 |                7.56 |
| Q08  | Digital   | multi_hop            | True         | True             | keyword        |               1    | True           | True              | True                | material description                     | Material Description                                             | SUPPORTED      |             0.619 |             1 |           0.914 |            8.875 |                9.79 |
| Q08  | Scanned   | multi_hop            | True         | True             | keyword        |               1    | True           | True              | True                | material description                     | Material Description                                             | SUPPORTED      |             0.624 |             1 |           0.903 |            9.329 |               10.23 |
| Q09  | Digital   | multi_hop            | True         | False            | keyword        |               0.33 | True           | True              | True                | supplier qualification                   | Supplier Qualification                                           | SUPPORTED      |             0.531 |             2 |           0.991 |            4.951 |                5.94 |
| Q09  | Scanned   | multi_hop            | True         | True             | keyword        |               0.67 | True           | True              | True                | supplier qualification                   | Supplier Qualification                                           | SUPPORTED      |             0.531 |             2 |           1.011 |            6.556 |                7.57 |
| Q10  | Digital   | vague_summary        | True         | True             | keyword        |               1    | True           | True              |                     | certificate of quality                   | Certificate Of Quality, Chain Of Custody, Supplier Qualification | SUPPORTED      |             0.129 |             4 |           1.151 |           13.182 |               14.33 |
| Q10  | Scanned   | vague_summary        | True         | True             | keyword        |               1    | True           | True              |                     | certificate of quality, chain of custody | Certificate Of Quality, Chain Of Custody, Supplier Qualification | SUPPORTED      |             0.129 |             4 |           0.875 |           14.693 |               15.57 |
| Q11  | Digital   | unanswerable_control | False        | True             | abstention     |             nan    | False          |                   |                     | -                                        | Certificate Of Quality                                           | UNSUPPORTED    |             0.339 |             2 |           0.94  |            1.043 |                1.98 |
| Q11  | Scanned   | unanswerable_control | False        | True             | abstention     |             nan    | False          |                   |                     | -                                        | Certificate Of Quality                                           | UNSUPPORTED    |             0.339 |             2 |           1.204 |            1.238 |                2.44 |
| Q12  | Digital   | unanswerable_control | False        | True             | abstention     |             nan    | False          |                   |                     | -                                        | Supplier Qualification                                           | UNSUPPORTED    |             0.344 |             2 |           0.992 |            1.134 |                2.13 |
| Q12  | Scanned   | unanswerable_control | False        | True             | abstention     |             nan    | False          |                   |                     | -                                        | Supplier Qualification                                           | UNSUPPORTED    |             0.344 |             2 |           0.963 |            1.135 |                2.1  |
| Q13  | Digital   | unanswerable_control | False        | True             | abstention     |             nan    | False          |                   |                     | -                                        | Bse/Tse Declaration                                              | UNSUPPORTED    |             0.33  |             1 |           1.085 |            1.135 |                2.22 |
| Q13  | Scanned   | unanswerable_control | False        | True             | abstention     |             nan    | False          |                   |                     | -                                        | Bse/Tse Declaration                                              | UNSUPPORTED    |             0.33  |             1 |           1.102 |            0.915 |                2.02 |
| Q14  | Digital   | unanswerable_control | False        | True             | abstention     |             nan    | False          |                   |                     | -                                        | Chain Of Custody                                                 | UNSUPPORTED    |             0.457 |             1 |           1.043 |            0.932 |                1.98 |
| Q14  | Scanned   | unanswerable_control | False        | True             | abstention     |             nan    | False          |                   |                     | -                                        | Chain Of Custody                                                 | UNSUPPORTED    |             0.457 |             1 |           1.056 |            0.956 |                2.01 |
| Q15  | Digital   | factual_lookup       | True         | True             | keyword        |               0.67 | True           | True              | True                | certificate of quality                   | Certificate Of Quality                                           | SUPPORTED      |             0.567 |             2 |           1.238 |            2.6   |                3.84 |
| Q15  | Scanned   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | certificate of quality                   | Certificate Of Quality                                           | SUPPORTED      |             0.542 |             3 |           0.993 |            3.755 |                4.75 |
| Q16  | Digital   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | cover letter                             | Certificate Of Quality, Cover Letter                             | SUPPORTED      |             0.061 |             3 |           1.113 |            2.545 |                3.66 |
| Q16  | Scanned   | factual_lookup       | True         | True             | keyword        |               1    | True           | True              | True                | cover letter                             | Certificate Of Quality, Cover Letter                             | SUPPORTED      |             0.118 |             4 |           0.869 |            2.522 |                3.39 |
| Q17  | Digital   | multi_hop            | True         | False            | keyword        |               0    | False          |                   |                     | -                                        | Certificate Of Quality                                           | UNSUPPORTED    |             0.283 |             2 |           0.991 |            1.151 |                2.14 |
| Q17  | Scanned   | multi_hop            | True         | False            | keyword        |               0    | False          |                   |                     | -                                        | Certificate Of Quality                                           | UNSUPPORTED    |             0.242 |             3 |           1.306 |            1.294 |                2.6  |

### By question type

| Category             | Variant   |   Runs |   Answer correct |   Citations correct |   Avg response (s) |
|:---------------------|:----------|-------:|-----------------:|--------------------:|-------------------:|
| compliance           | Digital   |      2 |             0.5  |                   1 |               4.07 |
| compliance           | Scanned   |      2 |             0.5  |                   1 |               4.3  |
| factual_lookup       | Digital   |      7 |             1    |                   1 |               3.84 |
| factual_lookup       | Scanned   |      7 |             1    |                   1 |               4.38 |
| multi_hop            | Digital   |      3 |             0.33 |                   1 |               5.96 |
| multi_hop            | Scanned   |      3 |             0.67 |                   1 |               6.8  |
| unanswerable_control | Digital   |      4 |             1    |                 nan |               2.08 |
| unanswerable_control | Scanned   |      4 |             1    |                 nan |               2.14 |
| vague_summary        | Digital   |      1 |             1    |                 nan |              14.33 |
| vague_summary        | Scanned   |      1 |             1    |                 nan |              15.57 |

## Routing ablation

| Mode          |   Questions |   Hit Rate |   Precision@K |   MRR |   Mean chunks |   Mean total (s) |
|:--------------|------------:|-----------:|--------------:|------:|--------------:|-----------------:|
| auto_route    |           4 |          1 |          0.25 |     1 |          1.75 |            8.965 |
| global_search |           4 |          1 |          0.25 |     1 |          4    |            7.205 |
| manual_filter |           3 |          1 |          0.25 |     1 |          1    |            6.4   |

### Detail

| ID   | Mode          | Hit   |   Precision@K |   RR | Sources used                                                                           |   Chunks used |   Mean similarity |   Retrieval (s) |   Generation (s) |   Total (s) |
|:-----|:--------------|:------|--------------:|-----:|:---------------------------------------------------------------------------------------|--------------:|------------------:|----------------:|-----------------:|------------:|
| Q01  | auto_route    | True  |          0.25 |    1 | Cover Letter                                                                           |             1 |             0.827 |           0.943 |            1.588 |        2.53 |
| Q01  | global_search | True  |          0.25 |    1 | Certificate Of Quality, Cover Letter, Material Description                             |             4 |             0.632 |           0.007 |            2.225 |        2.23 |
| Q01  | manual_filter | True  |          0.25 |    1 | Cover Letter                                                                           |             1 |             0.827 |           0.008 |            1.635 |        1.64 |
| Q04  | auto_route    | True  |          0.25 |    1 | Bse/Tse Declaration                                                                    |             1 |             0.487 |           1.008 |            0.92  |        1.93 |
| Q04  | global_search | True  |          0.25 |    1 | Bse/Tse Declaration, Chain Of Custody, Packaging Specification, Supplier Qualification |             4 |             0.229 |           0.007 |            1.729 |        1.74 |
| Q04  | manual_filter | True  |          0.25 |    1 | Bse/Tse Declaration                                                                    |             1 |             0.487 |           0.01  |            1.11  |        1.12 |
| Q08  | auto_route    | True  |          0.25 |    1 | Material Description                                                                   |             1 |             0.619 |           1.188 |            8.173 |        9.36 |
| Q08  | global_search | True  |          0.25 |    1 | Certificate Of Quality, Cover Letter, Material Description                             |             4 |             0.523 |           0.007 |           12.019 |       12.03 |
| Q08  | manual_filter | True  |          0.25 |    1 | Material Description                                                                   |             1 |             0.619 |           0.025 |           16.415 |       16.44 |
| Q10  | auto_route    |       |        nan    |  nan | Certificate Of Quality, Chain Of Custody, Supplier Qualification                       |             4 |             0.129 |           2.117 |           19.925 |       22.04 |
| Q10  | global_search |       |        nan    |  nan | Certificate Of Quality, Chain Of Custody, Supplier Qualification                       |             4 |             0.129 |           0.007 |           12.81  |       12.82 |

## Time by pipeline stage

| Stage              |   Total (s) |   Calls |   Avg (s/call) |
|:-------------------|------------:|--------:|---------------:|
| answer_generation  |      125.13 |      34 |          3.68  |
| routing            |       67.82 |      68 |          0.997 |
| ocr                |       65.38 |      13 |          5.029 |
| llm_judge          |       16.83 |      34 |          0.495 |
| classification     |       11.51 |      25 |          0.461 |
| boundary_detection |        5.75 |      22 |          0.261 |
| retrieval          |        0.55 |     136 |          0.004 |
| embedding          |        0.34 |       4 |          0.084 |
| indexing           |        0.02 |       4 |          0.004 |

## Full answers

**Q01 [Digital]: What is the recommended storage temperature for AKTA ready flow kits?**  
*File: pharma-blob-sample.pdf | scope: routed:Cover Letter*

The recommended storage temperature for AKTA ready flow kits is > +5°C. [From Cover Letter, Pages 1-1]

**Q01 [Scanned]: What is the recommended storage temperature for AKTA ready flow kits?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Cover Letter*

The recommended storage temperature for AKTA ready flow kits is > +5°C. [From Cover Letter, Pages 1-1]

**Q02 [Digital]: What is the lot number and expiration date for the AKTA ready Low Flow Kit?**  
*File: pharma-blob-sample.pdf | scope: routed:Certificate Of Quality*

[From Certificate Of Quality, Pages 2-2, Source: pharma-blob-sample.pdf]
Lot Number: 18356721
Expiration Date: 20260315

**Q02 [Scanned]: What is the lot number and expiration date for the AKTA ready Low Flow Kit?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Certificate Of Quality*

[From Certificate Of Quality, Pages 2-2, Source: pharma-blob-sample_scanned.pdf]
Lot Number: 18356721
[From Certificate Of Quality, Pages 2-2, Source: pharma-blob-sample_scanned.pdf]
Expiration Date: 20260315

**Q03 [Digital]: What material is used for the blister tray in the packaging assembly?**  
*File: pharma-blob-sample.pdf | scope: routed:Packaging Specification*

The material used for the blister tray in the packaging assembly is PETG. [From Packaging Specification, Pages 4-4]

**Q03 [Scanned]: What material is used for the blister tray in the packaging assembly?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Packaging Specification*

The material used for the blister tray in the packaging assembly is PETG. [From Packaging Specification, Pages 4-4]

**Q04 [Digital]: Does this product contain any materials of animal origin?**  
*File: pharma-blob-sample.pdf | scope: routed:Bse/Tse Declaration*

The provided documents do not contain enough information to answer this question.

**Q04 [Scanned]: Does this product contain any materials of animal origin?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Bse/Tse Declaration*

The provided documents do not contain enough information to answer this question.

**Q05 [Digital]: What sterilization methods are compatible with the AKTA ready Gradient Flow Section, and is EtO validated?**  
*File: pharma-blob-sample.pdf | scope: routed:Material Description*

[From Material Description, Pages 7-7]
Sterilization Methods Compatible: Autoclave, Gamma Irradiation
EtO Sterilization Not Validated

**Q05 [Scanned]: What sterilization methods are compatible with the AKTA ready Gradient Flow Section, and is EtO validated?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Material Description*

[From Material Description, Pages 7-7] The AKTA ready Gradient Flow Section is compatible with Autoclave at 121°C for 15 minutes and Gamma Irradiation at 25-40 kGy. EtO Sterilization is not validated.

**Q06 [Digital]: What ISO certifications does the supplier Cytiva Sweden AB hold, and are they still valid?**  
*File: pharma-blob-sample.pdf | scope: routed:Supplier Qualification*

The supplier Cytiva Sweden AB holds the following ISO certifications, and they are still valid:

- ISO 9001:2015, Valid Until: 31 December 2024, [From Supplier Qualification, Pages 8-8]
- ISO 13485:2016, Valid Until: 31 December 2024, [From Supplier Qualification, Pages 8-8]

**Q06 [Scanned]: What ISO certifications does the supplier Cytiva Sweden AB hold, and are they still valid?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Supplier Qualification*

The supplier Cytiva Sweden AB holds the following ISO certifications, and they are still valid:

- ISO 9001:2015 - [From Supplier Qualification, Pages 8-8]
- ISO 13485:2016 - [From Supplier Qualification, Pages 8-8]

Both certifications have validity until December 31, 2024.

**Q07 [Digital]: Where are the AKTA ready flow kit assemblies manufactured, and what is the distribution flow?**  
*File: pharma-blob-sample.pdf | scope: routed:Chain Of Custody*

The AKTA ready flow kit assemblies are manufactured at the Cytiva Eysins Facility in Switzerland. [From Chain Of Custody, Pages 10-10]

The distribution flow for the assemblies is as follows:
1. Raw materials received at Eysins facility (incoming QC)
2. Sub-assembly and final assembly (in-process checks)
3. Final quality release (per product-specific release criteria)
4. Shipped to Cytiva Distribution Center, Marlborough, MA
5. Distributed to end customer upon order

[From Chain Of Custody, Pages 10-10]

**Q07 [Scanned]: Where are the AKTA ready flow kit assemblies manufactured, and what is the distribution flow?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Chain Of Custody*

The AKTA ready flow kit assemblies are manufactured at the Cytiva Eysins Facility in Switzerland. [From Chain Of Custody, Pages 10-10]

The distribution flow for these assemblies is as follows:
1. Raw materials received at Eysins facility (incoming QC)
2. Sub-assembly and final assembly (in-process checks)
3. Final quality release (per product-specific release criteria)
4. Shipped to Cytiva Distribution Center, Marlborough, MA
5. Distributed to end customer upon order [From Chain Of Custody, Pages 10-10]

**Q08 [Digital]: What materials were used to construct the AKTA ready Gradient Flow Section, and are those materials confirmed free of animal-derived origin?**  
*File: pharma-blob-sample.pdf | scope: routed:Material Description*

The AKTA ready Gradient Flow Section was constructed using the following materials:
- Housing: Polypropylene (PP), [From Material Description, Pages 7-7]
- Gaskets: EPDM Rubber, [From Material Description, Pages 7-7]
- Pump Tubing: Platinum-Cured Silicone, [From Material Description, Pages 7-7]
- Inlet Fittings: PEEK, [From Material Description, Pages 7-7]
- Mixing Chamber: Borosilicate Glass, [From Material Description, Pages 7-7]

All components listed are confirmed free of animal-derived origin as they are specified as USP Class VI or ASTM standards, which typically indicate non-animal derived materials. [From Material Description, Pages 7-7]

**Q08 [Scanned]: What materials were used to construct the AKTA ready Gradient Flow Section, and are those materials confirmed free of animal-derived origin?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Material Description*

The AKTA ready Gradient Flow Section was constructed using the following materials:

- Housing: Polypropylene (PP), confirmed free of animal-derived origin [From Material Description, Pages 7-7].
- Gaskets: EPDM Rubber, confirmed free of animal-derived origin [From Material Description, Pages 7-7].
- Pump Tubing: Platinum-Cured Silicone, confirmed free of animal-derived origin [From Material Description, Pages 7-7].
- Inlet Fittings: PEEK, confirmed free of animal-derived origin [From Material Description, Pages 7-7].
- Mixing Chamber: Borosilicate Glass, confirmed free of animal-derived origin [From Material Description, Pages 7-7].

The provided documents do not contain enough information to determine if these materials are free of animal-derived origin.

**Q09 [Digital]: What packaging materials are used for part number 29477427, and is that part number on the approved product list for supplier Cytiva Sweden AB?**  
*File: pharma-blob-sample.pdf | scope: routed:Supplier Qualification*

The packaging material for part number 29477427 is lid film (1073B). [From Supplier Qualification, Pages 9-9]

Part number 29477427 is on the approved product list for supplier Cytiva Sweden AB. [From Supplier Qualification, Pages 8-8]

**Q09 [Scanned]: What packaging materials are used for part number 29477427, and is that part number on the approved product list for supplier Cytiva Sweden AB?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Supplier Qualification*

The packaging material for part number 29477427 is Tyvek, as it is listed under Sub-Supplier Material Provided with a qualification status of "Approved (2020)" in [From Supplier Qualification, Pages 9-9].

Part number 29477427 is indeed on the approved product list for supplier Cytiva Sweden AB, as it is listed under Approved Products with the description "High Flow Kit F, Modified, AKTA ready Flow Kit" in [From Supplier Qualification, Pages 8-9].

**Q10 [Digital]: Tell me about this product.**  
*File: pharma-blob-sample.pdf | scope: global (routing confidence below threshold)*

The product described is an AKTA ready High Flow Kit and an AKTA ready Low Flow Kit. Both kits are manufactured in compliance with ISO 9001 certified quality management system and are manufactured in Cytiva Eysins Facility, Eysins, Switzerland. The High Flow Kit, AKTA ready (Lot Number: 15102934) has an expiration date of July 12, 2021, while the Low Flow Kit, AKTA ready (Lot Number: 18356721) has an expiration date of March 15, 2026. These kits are shipped from Cytiva Eysins Facility to Cytiva Distribution Center, Marlborough, MA, USA, and then distributed to end customers upon orders. [From Certificate Of Quality, Pages 3-3, Source: pharma-blob-sample.pdf; From Certificate Of Quality, Pages 2-2, Source: pharma-blob-sample.pdf; From Chain Of Custody, Pages 10-10, Source: pharma-blob-sample.pdf]

**Q10 [Scanned]: Tell me about this product.**  
*File: pharma-blob-sample_scanned.pdf | scope: global (routing confidence below threshold)*

The product described is a High Flow Kit, AKTA ready, with the following details:

- [From Certificate Of Quality, Pages 2-2] The product is manufactured in compliance with ISO 9001 certified quality management system.
- [From Certificate Of Quality, Pages 2-2] It has a Lot Number: 15102934 and an Expiration Date: 20210712.
- [From Certificate Of Quality, Pages 2-2] The Product Description is: High Flow Kit, AKTA ready.
- [From Certificate Of Quality, Pages 2-2] The Product Release Criteria include Autoclave - Pump tubing test, Gamma Irradiation - Inlets test, Flow Rate Test, Pressure Integrity Test, Visual Inspection, and Package Integrity test.
- [From Chain Of Custody, Pages 10-10] The product follows a chain of custody from Eysins facility in Switzerland to Cytiva Distribution Center, Marlborough, MA, and then to the end customer.
- [From Chain Of Custody, Pages 10-10] Each assembly carries a unique lot number traceable to raw material lots, manufacturing date, and operator records.

**Q11 [Digital]: What is the lot number and expiration date for part number 29477427?**  
*File: pharma-blob-sample.pdf | scope: routed:Certificate Of Quality*

The provided documents do not contain enough information to answer this question.

**Q11 [Scanned]: What is the lot number and expiration date for part number 29477427?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Certificate Of Quality*

The provided documents do not contain enough information to answer this question.

**Q12 [Digital]: What was the outcome of the FDA inspection conducted in 2019?**  
*File: pharma-blob-sample.pdf | scope: routed:Supplier Qualification*

The provided documents do not contain enough information to answer this question.

**Q12 [Scanned]: What was the outcome of the FDA inspection conducted in 2019?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Supplier Qualification*

The provided documents do not contain enough information to answer this question.

**Q13 [Digital]: What is the home address of the Quality Assurance Director who signed the BSE/TSE declaration?**  
*File: pharma-blob-sample.pdf | scope: routed:Bse/Tse Declaration*

The provided documents do not contain enough information to answer this question.

**Q13 [Scanned]: What is the home address of the Quality Assurance Director who signed the BSE/TSE declaration?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Bse/Tse Declaration*

The provided documents do not contain enough information to answer this question.

**Q14 [Digital]: How many units of the AKTA ready Low Flow Kit were shipped to customers in the last fiscal quarter?**  
*File: pharma-blob-sample.pdf | scope: routed:Chain Of Custody*

The provided documents do not contain enough information to answer this question.

**Q14 [Scanned]: How many units of the AKTA ready Low Flow Kit were shipped to customers in the last fiscal quarter?**  
*File: pharma-blob-sample_scanned.pdf | scope: routed:Chain Of Custody*

The provided documents do not contain enough information to answer this question.

**Q15 [Digital]: What is the lot number and expiration date for the AKTA ready Gradient Flow Section With Inlets?**  
*File: sample-sdf-document.pdf | scope: routed:Certificate Of Quality*

[From Certificate Of Quality, Pages 2-2]  
Lot Number: 17242818  
Expiration Date: 20230126

**Q15 [Scanned]: What is the lot number and expiration date for the AKTA ready Gradient Flow Section With Inlets?**  
*File: sample-sdf-document_scanned.pdf | scope: routed:Certificate Of Quality*

The lot number for the AKTA ready Gradient Flow Section With Inlets is 17242818.
The expiration date is 20230126.
[From Certificate Of Quality, Pages 2-2, Source: sample-sdf-document_scanned.pdf]

**Q16 [Digital]: Who signed the cover letter, and what is their role?**  
*File: sample-sdf-document.pdf | scope: global (routing confidence below threshold)*

[From Cover Letter, Pages 1-1, Source: sample-sdf-document.pdf]
Signature: Mike Toner
Role: Product Manager

**Q16 [Scanned]: Who signed the cover letter, and what is their role?**  
*File: sample-sdf-document_scanned.pdf | scope: global (routing confidence below threshold)*

[From Cover Letter, Pages 1-1, Source: sample-sdf-document_scanned.pdf]
- Signed by: Wi Mike Toner
- Role: Product Manager

**Q17 [Digital]: What lot numbers appear on the certificates of quality in this document?**  
*File: sample-sdf-document.pdf | scope: routed:Certificate Of Quality*

The provided documents do not contain enough information to answer this question.

**Q17 [Scanned]: What lot numbers appear on the certificates of quality in this document?**  
*File: sample-sdf-document_scanned.pdf | scope: routed:Certificate Of Quality*

The provided documents do not contain enough information to answer this question.

