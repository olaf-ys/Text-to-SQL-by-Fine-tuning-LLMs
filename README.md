# Business Applications of Text-to-SQL by Fine-tuning LLMs
Yuanshan Zhang\
Questrom School of Business, Boston University\
BA 890: Analytics Practicum\
Dr. Ritwick Roy\
August 15, 2024

## 1. Introduction
While LLMs (Large Language Models) are revolutionizing the entire industry, one of their major influences is lowering the technical bar to perform data analysis. For instance, one only needs to communicate with the model in natural language to get visualizations. Likewise, many data tasks can be automated. Querying databases using SQL, for instance, is a crucial intermediate step of data analysis that has been most frequently performed by human analysts in the past decades but is now in urgent demand to be automated by LLMs because generating SQL with LLMs not only reduces human labor but also empowers non-technicians to access the data.

A simple way for businesses to achieve the technique is to use LLMs’ APIs from service providers such as OpenAI, Google Cloud Platform, Amazon Web Services, etc. These models are accurate and sufficient for translating natural language into SQL across a broad and general context and semantics. However, they might fail or produce inaccurate answers for specialized tasks. For instance, in calculating financial technical indices, a question such as ‘What is the RSI for GOOG’ might lead to errors if the model lacks specific financial knowledge. Of course, advanced LLMs such as GPT and Bard know what RSI (Relative Strength Index) is, but they might not be up-to-date with the latest dynamics in the financial industry. A solution for this problem is to fine-tune LLMs and accommodate them to specialized tasks. In this paper, I will analyze the techniques, feasibility, application, and potential risks of fine-tuning LLMs for text-to-SQL generation by training the small version of Google’s T5 [1] on the WikiSQL dataset [2].
## 2. Data Description
WikiSQL is a dataset of 80,654 hand-annotated examples of questions and SQL queries distributed across 24,241 tables from Wikipedia.
In my analysis, ‘question’, ‘table-header’, and ‘table-types’ are used as model inputs, and ‘sql-human_readable’ is used as the label:
- question: the natural language question
- table
  - header: a list of column names in the table
  - types: the data type of each column
- sql
  - human_readable: labeled SQL queries

Ideally, features such as ‘table-rows’ and ‘table-caption’ should be included as model inputs, but the maximum sequence length allowed by the tokenizer of the small version of Google’s T5 is 512. So, to avoid truncation of the headers, these features are excluded.
## 3. Methodology
Generating SQL queries using LLMs is a form of TQA (Table Question Answering). TQA is an NLP (natural language processing) task that answers user questions based on tabular data. The alternative form is generating answers without intermediate steps such as translating answers into SQL queries. In business scenarios, generating SQL queries is more desirable and reliable not only because it outperforms the first approach in being better at answering more complex questions, but also because most businesses require analytic results to be trackable, and SQL caters to this requirement.

To empower LLM to translate questions into SQL queries, I first loaded the WikiSQL dataset and the small version of Google’s T5 from Hugging Face. Then, I preprocessed and tokenized the WikiSQL dataset. 

To demonstrate the effectiveness of fine-tuning, I determined the baseline for T5’s text-to-SQL generation performance using BLEU (Bilingual Evaluation Understudy), which measures the n-gram overlap between generated queries and actual queries. However, a single natural language query may correspond to multiple equivalent queries. For example, the order of fields and tables may differ while maintaining the same semantics. The BLEU metric may score these equivalent SQL queries lower because it only considers surface form matching. BLEU is adopted for simplicity, but in practice, execution accuracy and logical form accuracy should be used as the primary metrics. The baseline BLEU and loss for T5 are about 0.0145 and 12.9694, indicating that T5 outputs hallucinations rather than SQL queries.

T5 small has 6 million parameters, making it hard to train the entire model on a personal device. In such a situation, LoRA (Low-Rank Adaptation) considerably reduces the trainable parameters by decomposing some weight matrices into the product of 2 low-rank matrices (rank 8 is used in my research). For instance, an attention head contains 3 matrices: $W_Q$, $W_K$, and $W_V$. LoRA decomposes each of these matrices into two low-rank matrices such that $W_Q=A_QB_Q$ and updates only $A_Q$ and $B_Q$ during training. After applying LoRA to T5, trainable parameters are reduced to less than half of the original parameters.
4. Experiment Results

Table1
| Epoch | Training Loss | Validation Loss | BLEU    |
|-------|---------------|-----------------|---------|
| 1     | 0.201800      | 0.132348        | 0.480064|
| 2     | 0.136700      | 0.095944        | 0.553002|
| 3     | 0.120700      | 0.084832        | 0.574947|
| 4     | 0.112300      | 0.081770        | 0.582535|

Table 1 shows that both the training and validation loss steadily decrease during training, and in the last epoch, the training loss is slightly better than the validation loss, indicating a good fit. Moreover, the test loss is 0.0833 and the test BLEU is 0.5754, indicating a good generalization ability. Compared to the baseline loss (12.9694) and BLEU (0.0145), fine-tuned T5 is now significantly better at translating text into SQL queries.

Table 2
| Question | Generated SQL Command | Label SQL Command |
|----------|-----------------------|-------------------|
| What is the black Caribbean population when the black African population is less than 10552.0? | `SELECT Black Caribbean Population FROM table WHERE Black Caribbean Population = 10552.0` | `SELECT MIN Black Caribbean Population FROM table WHERE Black African Population < 10552.0` |
| How many tv series had an adult rating of 1.2? | `SELECT MIN TV season FROM table WHERE Adult rating = 1.2` | `SELECT COUNT TV season FROM table WHERE Rating (adults 18-49) = 1.2` |
| What's the percentage in 2001 in Manitoba where the percentage in 2011 is 0.8%? | `SELECT % 2001 FROM table WHERE % 2011 = 0.8%` | `SELECT % 2001 FROM table WHERE % 2011 = 0.8% AND Province = Manitoba` |

However, some common mistakes are made by the model as shown in Table 2. In the first example, the ‘MIN’ aggregation operator may be unnecessary but the model used the wrong logical operator ‘=’ instead of ‘<’ since ‘<’ rarely appears in the labels. In the second example, the model used the wrong aggregation operator and misspelled the header’s name, which might be fixed by fine-tuning hyperparameters such as increasing the training epochs. In the 3rd example, the model failed to use ‘AND’ to connect condition 1 and condition 2 since most labels contain a single condition.
5. Future Work
Future work to refine the model includes: 1. Tune the model using datasets with more complex query labels that incorporate window functions, subqueries, CTEs (Common Table Expression), table joins, etc. 2. Add detailed schema and data description to model inputs. 3. Use LLMs with higher capacity 4. Experiment with different combinations of hyper-parameters. 5. Use better evaluation metrics such as execution accuracy and logical form accuracy.
6. Business Applications
Text-to-SQL generation has many real-world business applications. In a B2C scenario such as mobile banking, it enables customers to access their savings or spending more easily without getting lost due to complex product designs, thereby improving service quality. Within an organization, it enables a manager to access data by delivering his ideas directly and precisely to a model rather than a data analyst. Therefore, adopting the technique will not only significantly reduce the communication cost, but also offer the decision-makers better control of business operations.

To successfully implement text-to-SQL generation in work production, a company first needs to have an extensive database and provide detailed schema and data descriptions. Second, form a large enough training dataset using questions, table information, and query records. Then, allocate resources to fine-tune the model. Moreover, develop rigorous testing methods such as executing the queries in the actual database to calculate accuracy. Finally, keep refining and updating the model with new data.
7. Conclusion
In this paper, I demonstrated how to use LLMs to generate SQLs by adding the LoRA adapter to the small version of Google’s T5 and fine-tuning the model on the WikiSQL dataset. Compared to the baseline, fine-tuned T5’s performance significantly improved, confirming the effectiveness of fine-tuning pre-trained LLMs. In real-world business applications, text-to-SQL generation not only improves service quality but also reduces the communication cost. On the one hand, companies that have limited resources or little demand for specialized context and semantics can use LLMs’s APIs. On the other hand, businesses with resources and demand for specialized context and semantics can develop their own models by fine-tuning pre-trained LLMs.

## References
1. Raffel, C., Shazeer, N., Roberts, A., Lee, K., Narang, S., Matena, M., ... & Liu, P. J. (2020). Exploring the limits of transfer learning with a unified text-to-text transformer. J. Mach. Learn. Res., 21(140), 1-67.

2. Victor Zhong, Caiming Xiong, and Richard Socher. 2017. Seq2SQL: Generating Structured Queries from Natural Language using Reinforcement Learning.



