# Semantic segmentation of natural language texts using named entity recognition
### Abstract
This paper is devoted to the creation of an algorithm for automatic
segmentation and classification of texts in natural language, based on the
method of recognizing named entities.

# 1. Introduction
The task of text segmentation and classification (Topic-aware text segmentation, TATS) is to split the source text into smaller, non-overlapping, semantically
coherent text units called segments and assign each of the segments a single
class from a predefined set.

All existing approaches to solving the problem of text segmentation can be classified into two main types. The first type includes methods focused solely on segmentation. Such approaches are considered insufficiently effective for this task, since they require the use of a separate model to perform classification, which increases the vulnerability of the system to changes in data and complicates the deployment of the solution in real-world operating conditions. The second type involves creating a single model that simultaneously performs segmentation and classification. Such models are easier to implement and use in practice, they have greater stability and predictability. However, their disadvantage is the high complexity of the neural network architecture, which leads to high computational resource costs and makes it more difficult to make change or improvements to the solution.

Thus, an important problem that is being solved in this work is the creation of models of the second type, which would be much simpler than existing approaches in terms of architecture, but would have comparable segmentation and classification quality.

The novelty of the research lies in the fact that an approach to solving the TATS task based on solving NER (Named Entity Recognition) task is proposed.
## 1.1 Team
- Shulgin Egor Preparation of the project architecture idea, analysis of existing approaches, creation of the model based on the perceptron architecture, preparation of the report.
- Vorsin Egor Analysis of existing approaches, creation of a data processing pipeline, experiments with adding LSTM layers to the model, preparation of the repository.

# 2. Model Description
In the beginning, we will prove why the TATS problem can be solved through the NER problem. Let us look at the formulations of both tasks. For the TATS task: divide the text into disjoint semantically related parts and assign each of these parts a class from a predefined set. For the NER task: select from the text the parts semantically related to a certain class from a predefined set. The equivalence of statements is achieved if we perceive segments as named entities and require the markup of the text to continuously fill in the entire text.

To achieve continuous markup, we abandoned the classic B-, I-, O- markup of the text, where B- (Begin) means the token of the beginning of the segment, I- (Inner) means the token of the inner part of the segment, and O (-Outer) means the token lying outside the segments. Since any token within our formulation lies inside a segment, we used the markup B-, I-, and L-, where L- (Last) means the token of the end of the segment, and B- and I- retain their previous meaning.

| The | first | segment. | The | second | segment. |
| ---- | ---- | ---- | ---- | ---- | ---- |
| B-cls1 | I-cls1 | L-cls1 | B-cls2 | I-cls2 | L-cls2 |


We used the token classification method to solve the NER problem, i.e. for each token, its markup (B-, I-, or L-) and its topic were simultaneously predicted. The number of predicted classes nclasses is thus calculated as

*n<sub>classes</sub> = n<sub>topics</sub> · 3*
, where n<sub>topics</sub> is the number of unique topics in the data set.

Two approaches to the model architecture were tested: based on a perceptron and using an LSTM layer. The perceptron-based model is designed as follows:
- Tokenized source text is sent to the BERT[?] encoder model. The output of it is a vector representation of the text *embeddings*.
- Then we used a fully connected neural network, which consisted of combinations of linear maps; LayerNormalization function; activation functions and Dropout function with parameter p = 0.3, the embedding dimension is reduced to the number of classes, that is, the output is the dimension of len_context × n_classes. To solve this problem, the activation function GELU was used
where Φ(x) is the distribution function of the standard normal distribution. Then we assume the function feed-forward network as a composition of alternating linear layers, LayerNormalization, activation functions mentioned above and Dropout. 
- Using the multi-variable logistic function *Softmax* for the vector *x* we reduced the real outputs from the previous stage to values from 0 to 1, thus obtaining the probabilities of each value. By applying this function to each token, we
obtain the probability distribution of the token belonging to a certain class.<br>
The second architecture differs from the perceptron only in that an LSTM layer was added between the embedding layer and the first linear layer. All other parameters were inherited from the previous model.<br>
Since the classes are strongly unbalanced, the focal loss function with the parameter γ = 2 was used as a loss function instead of cross-entropy.<br>
We reduced the real outputs from the previous stage to values from 0 to 1, thus obtaining the probabilities of each value. By applying this function to each token, we obtain the probability distribution of the token belonging to a certain class<br>

# 3. Dataset
WikiSection [en_disease](https://github.com/sebastianarnold/WikiSection), consisting of Wikipedia articles in English on the subject of medicine and available for the research purposes, was used as a dataset.The breakdown into training, validation and test samples is made in the proportions of 70%/10%/20%.

# 5. Experiments
## 5.1 Metrics
**For the task of segmentation** we used Pk metric proposed in the [?] article on algorithmic text segmentation. To measure it, you need to walk through the

| Language | English |
| ---- | ---- |
| Number of texts | 3590 |
| Average text length | 1100 |
| Average number of segments in text | 8 |
| Number of classes | 27 |

text with a sliding window, checking each step whether the sides of the window are in different segments in the true and predicted layout. If the values are different, then add 1 to the counter. At the end of the pass, the counter is normalized by dividing by the number of steps of the sliding window. Thus, the metric value is equal to the fraction of window steps in which the position of its sides differed for the two layouts. The smaller the metric, the better the text segmentation is performed.

**For the classification problem**, the standard metric F1 with microaveraging will be used, equal to the average harmonic accuracy and completeness.

Since the predicted segmentation is not perfect and generates parts of the text that do not match the true ones, the segment correlation scheme described in the article [?] is used to obtain pairs of class labels:
1. First, all segments of the true markup correspond to the predicted segments at the largest intersection.
2. Then the remaining unmarked parts of the predicted segmentation are correlated with the true markup at the largest intersection.

# 5.2 Experiment Setup
AdamW was used as an optimizer, the learning rate was set at 5·10−5, the focal loss function’s parameter γ was set at γ = 2.

The final perceptron model was trained on 3 epochs.

# 6. Results
The perceptron model based on the named entity recognition approach surpasses all approaches except SOTA Tipster in terms of segmentation quality on this dataset and is comparable to all models in terms of classification quality, although it is inferior to them in terms of F1 metric.

| Model / Metrics | P<sub>k</sub> | F1 |
| ---- | ---- | ---- |
| SECTOR | 26.3 | 55.8 |
| S-LSTM | 20.0 | 59.3 |
| Tipster (SOTA) | 14.2 | 62.2 |
| NER (perceptron)  | 19.9 | 54.2 |
| NER (with LSTM)  | 20.5 | 56.9 |

# 7. Conclusion
An algorithm for solving the TATS problem based on solving the equivalent formulation of the NER problem has been developed. This algorithm demonstrates comparable segmentation and classification quality with existing solutions, being significantly more computationally efficient due to a simpler architecture.
