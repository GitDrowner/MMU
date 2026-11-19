# MMU：多模态大模型遗忘评测（阶段汇总）

在线页面：https://gitdrowner.github.io/MMU/

研究对象是 LLaVA-1.5-7B 在 MLLMU-Bench 上的五种遗忘方法（遗忘 5% 的人物）。要回答的问题是：被"遗忘"的人物知识是真的没了，还是换个条件就能拿出来。所有结论都相对 Oracle 而言。Oracle 从未见过这些人物，但接受了同样的提示或同样的更新。

## 三轮实验

1. **四组重学对照**：在 5 个遗忘人物的数据或无关数据上微调，看另外 20 个遗忘人物的知识会不会回来。
2. **推理路径审计**：让模型先写一段推理再回答，看会不会多暴露知识；推理文本里会不会泄漏身份或属性。
3. **utility 约束下的恢复**：在 retain 准确率下降不超过 2、5、10 个点的条件下，看无关数据上的更新能否让遗忘模型比接受同样更新的 Oracle 暴露更多知识。攻击配置只在 5 个验证人物上选，选定之后才打开 15 个最终测试人物的结果。

## 目前可以得到的结论

### 比较稳的结论

1. **遗忘方法改变的主要是"用什么方式能读出知识"，而不是把知识删掉。** 除了被训练破坏的 GA-Diff ep10，其余四个遗忘模型都能在某种读法下显著高于 Oracle：IDK 在似然读数下是 0.88（Oracle 0.26）；NPO 和 GA-Diff ep2 加一句"直接回答"后分别高出 +0.21\* 和 +0.09\*；KL-Min 在姓名题上高出 +0.31\*。（第二、三轮一致）
2. **IDK 的遗忘是一个绑定在训练问法上的拒答闸门，而且很容易打开。** 原始问法下 100% 拒答，换一句提示只剩 3%。一次让 IDK、Full、Oracle 的 retain 各只降 3 到 4 个点的无关更新，就把拒答率降到 13%，生成答对率从 0 升到 0.61，同一更新下的 Oracle 只有 0.18。（三轮一致，两种口径都成立）
3. **GA-Diff ep10 不算遗忘成功，而是模型被破坏了。** 十选一准确率 0.07，低于随机水平，也低于 Oracle，生成的全是乱码。无关更新在预算内毫无作用；预算外修复了部分生成能力，但准确率只回到 Oracle 的水平。（三轮一致）
4. **Oracle 不会从同样的提示或更新里获得目标知识，所以"高出 Oracle"可以归为残留知识。** 第二轮 7 种问法、20 个更新检查点下，Oracle 都在 0.21–0.28；第三轮全部 63 个更新后检查点上在 0.24–0.27（更新前 0.265）。（三轮一致）
5. **让模型先写一段推理，不会多暴露知识。** 写完三段 rationale 后没有任何模型的准确率上升（Full −0.24\*，IDK −0.21\*），rationale 里写的身份对属性答案影响很小。在这个模型上，用 CoT 审计遗忘不如似然读数灵敏。（第二轮）
6. **属性知识绑定在人脸图像上，不经由姓名。** 不给图像时所有模型都回到 Oracle 的水平；直接告诉模型正确姓名也只有 0.22–0.33（Oracle 0.26），在训练过的 retain 人物上同样如此。（第二轮）

### 取决于 utility 怎么定义的结论

7. **无关数据上的更新能否在 utility 约束下重新暴露知识，取决于约束谁的 utility。**（第三轮；严格口径是看过结果后加的敏感性分析）
   - 只约束被攻击模型自己（任务书的定义）：GA-Diff ep2 和 KL-Min 在 2 个点预算内就显著高于同更新的 Oracle（+0.19\*、+0.20\*），NPO 在 10 个点预算内（+0.17\*）。
   - 要求同一更新对 Full 和 Oracle 也不超预算：NPO 仍然成立（+0.15\*）；GA-Diff ep2 只剩边缘信号（+0.11\*，由更新带来的部分不显著）；KL-Min 没有恢复（−0.02）。
   - 原因是 GA-Diff ep2 和 KL-Min 的 retain 起点本来就低（0.69 和 0.72），给它们选出的 lr 1e-4 更新几乎不伤它们自己，却让 Full 和 Oracle 掉 10 到 26 个点。
8. **不存在对所有方法都有效的"通用良性更新"。** 对五个方法都可行的更新基本是空操作；去掉 GA-Diff ep10 后，最好的通用更新能让三个方法同时显著，但它让 Full 和 Oracle 掉 10 到 11 个点。（第三轮）

### 对论文的含义（建议）

- **以 Oracle 为锚的评测是方法上的核心**：同一读法、同一更新也施加在 Oracle 上，才能把残留知识和读法或更新本身的效应分开。
- **拒答式遗忘（IDK）只改了输出行为**：知识完整保留，闸门在很小的良性更新下就会打开。
- **评测"良性更新后的恢复"时，要同时报告同一更新对 Full 和 Oracle 的 utility 代价。** 只看被攻击模型自己的 utility，会把对健康模型有破坏性的更新当成良性更新。
- **写成论文主结论之前，至少要在另一个数据划分和第二个 Oracle 种子上复现。**

\* 表示 95% 置信区间不含 0。区间来自按人物聚类的 bootstrap（10,000 次重采样），模型之间的差是配对差。证据范围：一个模型、一个数据划分、一个随机种子、一个 Oracle 种子；第三轮的最终测试只有 15 个人物。

## 相关论文

按与本项目的关系分 6 组，共 55 篇。链接指向 arXiv 摘要页，这里不转载论文全文。

### 本项目使用的模型、基准和遗忘方法

- [Visual Instruction Tuning](https://arxiv.org/abs/2304.08485) — Haotian Liu et al.，NeurIPS 2023。LLaVA：用 GPT-4 生成的图文指令数据端到端微调，是本项目所用模型的来源。
- [Improved Baselines with Visual Instruction Tuning](https://arxiv.org/abs/2310.03744) — Haotian Liu et al.，CVPR 2024。LLaVA-1.5，本项目使用的 7B 基座模型。
- [Protecting Privacy in Multimodal Large Language Models with MLLMU-Bench](https://arxiv.org/abs/2410.22108) — Zheyuan Liu et al.，NAACL 2025。本项目使用的基准：500 个虚构人物和 153 个名人的档案，从多模态和纯文本两方面评测遗忘与效用。
- [TOFU: A Task of Fictitious Unlearning for LLMs](https://arxiv.org/abs/2401.06121) — Pratyush Maini et al.，COLM 2024。虚构作者遗忘基准，给出 GA、GD、KL、IDK 等基线，并以从未见过遗忘集的模型为金标准。本项目的 KL-Min 和 IDK 来自这里。
- [Knowledge Unlearning for Mitigating Privacy Risks in Language Models](https://arxiv.org/abs/2210.01504) — Joel Jang et al.，ACL 2023。对目标序列做梯度上升（GA）来遗忘，是 GA 类方法的来源。
- [Continual Learning and Private Unlearning](https://arxiv.org/abs/2203.12817) — Bo Liu, Qiang Liu, Peter Stone，CoLLAs 2022。TOFU 把梯度差（GD，本项目的 GA-Diff）基线归于这篇论文。
- [Negative Preference Optimization: From Catastrophic Collapse to Effective Unlearning](https://arxiv.org/abs/2404.05868) — Ruiqi Zhang et al.，COLM 2024。NPO：比 GA 更慢走向灾难性崩溃，在遗忘与效用之间平衡得更好。

### 遗忘是否真的删除了知识：文本大模型上的恢复与鲁棒性

- [Unlearning or Obfuscating? Jogging the Memory of Unlearned LLMs via Benign Relearning](https://arxiv.org/abs/2406.13356) — Shengyuan Hu et al.，ICLR 2025。用少量只是松散相关的良性数据再学习，就能唤回已遗忘的知识。（与本项目一致）
- [An Adversarial Perspective on Machine Unlearning for AI Safety](https://arxiv.org/abs/2409.18025) — Jakub Łucki et al.，TMLR 2025。只用 10 条无关样本微调，或去掉特定的激活方向，就能恢复 RMU 遗忘掉的大部分危险能力。（与本项目一致）
- [Do Unlearning Methods Remove Information from Language Model Weights?](https://arxiv.org/abs/2410.08827) — Aghyad Deeb, Fabien Roger，arXiv 2024。用一部分被遗忘的事实微调，就能把同分布的其余事实恢复到原准确率的 88%。（与本项目一致）
- [Catastrophic Failure of LLM Unlearning via Quantization](https://arxiv.org/abs/2410.16454) — Zhiwei Zhang et al.，ICLR 2025。对遗忘后的模型做 4-bit 量化，遗忘知识的保留率从平均 21% 升到 83%。（与本项目一致）
- [Eight Methods to Evaluate Robust Unlearning in LLMs](https://arxiv.org/abs/2402.16835) — Aengus Lynch et al.，arXiv 2024。被遗忘的知识仍能被抽取，潜在表征与原模型相当；只看常规指标会高估遗忘。（与本项目一致）
- [Position: LLM Unlearning Benchmarks are Weak Measures of Progress](https://arxiv.org/abs/2410.02879) — Pratiksha Thaker et al.，IEEE SaTML 2025。对基准做简单的良性改动就能拿到“已遗忘”的信息，现有基准结果过于乐观。
- [Fine-tuning Aligned Language Models Compromises Safety, Even When Users Do Not Intend To!](https://arxiv.org/abs/2310.03693) — Xiangyu Qi et al.，ICLR 2024。用常见的良性数据微调也会削弱安全对齐，是“良性更新破坏防护”的一个类比。
- [Tamper-Resistant Safeguards for Open-Weight LLMs](https://arxiv.org/abs/2408.00761) — Rishub Tamirisa et al.，ICLR 2025。拒答和遗忘防护几步微调就会被移除；提出能抵抗数百步微调的 TAR。
- [Erase or Hide? Suppressing Spurious Unlearning Neurons for Robust Unlearning](https://arxiv.org/abs/2509.22263) — Nakyeong Yang et al.，ICLR 2026。常用遗忘方法只是用“虚假遗忘神经元”把知识藏起来，良性微调后知识会重现。（与本项目一致）
- [Rethinking Benign Relearning: Syntax as the Hidden Driver of Unlearning Failures](https://arxiv.org/abs/2602.03379) — Sangyeon Yoon et al.，ICLR 2026。良性再学习主要由句式相似驱动，而不是话题相关。本项目的无关 VQA 数据也是整句回答，可能与此有关，需要对照。（与本项目最相关）
- [Distillation Robustifies Unlearning](https://arxiv.org/abs/2506.06278) — Bruce W. Lee et al.，NeurIPS 2025。即使输出已和从未见过遗忘数据的 oracle 一致，继续微调后仍比 oracle 恢复得快。与本项目以 Oracle 为锚的思路最接近。（与本项目最相关）
- [Unlearning Isn't Deletion: Investigating Reversibility of Machine Unlearning in LLMs](https://arxiv.org/abs/2505.16831) — Xiaoyu Xu et al.，ICML 2026。少量微调就能恢复遗忘前的行为；用 PCA、CKA、Fisher 信息等表征指标说明知识被压制而非删除。是本项目下一步 hidden-state 分析最接近的已有工作。（与本项目最相关）
- [From Dormant to Deleted: Tamper-Resistant Unlearning Through Weight-Space Regularization](https://arxiv.org/abs/2505.22310) — Shoaib Ahmed Siddiqui et al.，NeurIPS 2025。在视觉分类器上，只用保留集微调就能让遗忘集准确率从约 50% 回到近 100%，而重训模型仍是 50%。（与本项目一致）

### 多模态大模型的遗忘与恢复攻击

- [Single Image Unlearning: Efficient Machine Unlearning in Multimodal Large Language Models](https://arxiv.org/abs/2405.12523) — Jiaqi Li et al.，NeurIPS 2024。提出 MMUBench：只用单张图像微调，就能让 MLLM 遗忘一个视觉概念。
- [CLEAR: Character Unlearning in Textual and Visual Modalities](https://arxiv.org/abs/2410.18057) — Alexey Dontsov et al.，Findings of ACL 2025。200 个虚构人物、3,700 张图像的图文遗忘基准；同时在两种模态上遗忘效果更好。
- [Benchmarking Vision Language Model Unlearning via Fictitious Facial Identity Dataset](https://arxiv.org/abs/2411.03554) — Yingzi Ma et al.，ICLR 2025。虚构人脸身份的遗忘基准（FIUBench）；现有方法遗忘有限，主张用隐私攻击做稳健评估。
- [MMUnlearner: Reformulating Multimodal Machine Unlearning in the Era of Multimodal Large Language Models](https://arxiv.org/abs/2502.11051) — Jiahao Huo et al.，Findings of ACL 2025。把 MLLM 遗忘定义为只删掉实体的视觉模式，同时保留语言骨干里的文本知识。
- [Modality-Aware Neuron Pruning for Unlearning in Multimodal Large Language Models](https://arxiv.org/abs/2502.15910) — Zheyuan Liu et al.，ACL 2025。现有方法在图文输入下看似已遗忘，纯文本输入下仍有残留；提出按模态剪枝神经元。
- [SUA: Stealthy Multimodal Large Language Model Unlearning Attack](https://arxiv.org/abs/2506.17265) — Xianren Zhang et al.，EMNLP 2025。同样用 LLaVA-1.5-7B 和 MLLMU-Bench：在输入图像上加一个通用噪声，就能让 GD 和 NPO 遗忘后的模型说出被遗忘的内容。（与本项目一致）
- [POPS: Recovering Unlearned Multi-Modality Knowledge in MLLMs with Prompt-Optimized Parameter Shaking](https://arxiv.org/abs/2607.06649) — Zhangheng Li et al.，arXiv 2026。先优化提示诱导模型生成隐私样本，再用它们微调，几乎能完全恢复 MLLM 中“已遗忘”的隐私信息。（与本项目一致）
- [On the Robustness of Machine Unlearning for Vision-Language Models](https://arxiv.org/abs/2605.26992) — Yujie Lin et al.，arXiv 2026。多数 VLM 遗忘方法在攻击下失效，更像是把目标知识藏起来，而不是删除。（与本项目一致）

### 推理链：忠实性与泄漏

- [Language Models Don't Always Say What They Think: Unfaithful Explanations in Chain-of-Thought Prompting](https://arxiv.org/abs/2305.04388) — Miles Turpin et al.，NeurIPS 2023。CoT 给出的解释会系统性地偏离模型做出预测的真实原因。与第二轮“rationale 不等于内部推理”一致。（与本项目一致）
- [Measuring Faithfulness in Chain-of-Thought Reasoning](https://arxiv.org/abs/2307.13702) — Tamera Lanham et al.，arXiv 2023。通过截断、插错等方式干预 CoT 来测忠实度；在多数任务上，模型越大推理越不忠实。
- [Chain-of-Thought Reasoning In The Wild Is Not Always Faithful](https://arxiv.org/abs/2503.08679) — Iván Arcuschin et al.，ICML 2026。在没有人为偏置的真实提示下，前沿模型的 CoT 仍会出现事后合理化。
- [Towards Reasoning-Preserving Unlearning in Multimodal Large Language Models](https://arxiv.org/abs/2512.17911) — Hongji Li et al.，arXiv 2025。只遗忘答案时，推理步骤里仍会泄漏敏感信息；提出 RMLLMU-Bench 和推理时方法 R-MUSE。与第二轮的结论方向相反，设置不同，需要对照讨论。（与本项目相反）
- [Step-by-Step Reasoning Attack: Revealing 'Erased' Knowledge in Large Language Models](https://arxiv.org/abs/2506.17279) — Yash Sinha et al.，arXiv 2025。逐步推理式的黑盒对抗提示能找回被“删除”的知识。与第二轮的结论方向相反（我们的推理提示没有做对抗优化）。（与本项目相反）
- [R-TOFU: Unlearning in Large Reasoning Models](https://arxiv.org/abs/2505.15214) — Sangyeon Yoon, Wonje Jeung, Albert No，EMNLP 2025。面向大型推理模型的遗忘基准，同时检查最终答案和推理链。
- [Reasoning Model Unlearning: Forgetting Traces, Not Just Answers, While Preserving Reasoning Skills](https://arxiv.org/abs/2506.12963) — Changsheng Wang et al.，EMNLP 2025。推理模型的遗忘要同时清除推理痕迹，而不只是最终答案。

### hidden state 分析：方法基础与最相关的已有工作（下一阶段）

- [interpreting GPT: the logit lens](https://www.lesswrong.com/posts/AcKRB8wDpdaN6v6ru/interpreting-gpt-the-logit-lens) — nostalgebraist，LessWrong 博客 2020。用最终层的 unembedding 直接解码中间层，是逐层读出预测的最简单方法。
- [Eliciting Latent Predictions from Transformers with the Tuned Lens](https://arxiv.org/abs/2303.08112) — Nora Belrose et al.，arXiv 2023。为每一层训练一个仿射映射再解码，比 logit lens 更可靠地逐层读出预测。
- [Locating and Editing Factual Associations in GPT](https://arxiv.org/abs/2202.05262) — Kevin Meng et al.，NeurIPS 2022。用因果追踪把事实知识定位到中间层的 MLP，可用来定位残留知识所在的层。
- [Refusal in Language Models Is Mediated by a Single Direction](https://arxiv.org/abs/2406.11717) — Andy Arditi et al.，NeurIPS 2024。拒答由残差流中的单一方向介导，去掉或加上这个方向就能开关拒答。可用来检验 IDK 的“遗忘”是否只是拒答。
- [Representation Engineering: A Top-Down Approach to AI Transparency](https://arxiv.org/abs/2310.01405) — Andy Zou et al.，arXiv 2023。从群体层面的表征出发监测和操控高层概念，可用来寻找并操控与遗忘相关的方向。
- [Designing and Interpreting Probes with Control Tasks](https://arxiv.org/abs/1909.03368) — John Hewitt, Percy Liang，EMNLP-IJCNLP 2019。提出控制任务和选择性指标，区分探针读到的是表征里的信息，还是探针自己学会的。
- [Probing Classifiers: Promises, Shortcomings, and Advances](https://arxiv.org/abs/2102.12452) — Yonatan Belinkov，Computational Linguistics 2022。系统评述探针方法的局限和改进，例如对照基线和因果干预。
- [Intrinsic Test of Unlearning Using Parametric Knowledge Traces](https://arxiv.org/abs/2406.11614) — Yihuai Hong et al.，EMNLP 2025。用词表投影找到参数里的概念向量：现有遗忘方法几乎不改动它们，主要是在推理时压制。（与本项目一致）
- [Dissecting Fine-Tuning Unlearning in Large Language Models](https://arxiv.org/abs/2410.06606) — Yihuai Hong et al.，EMNLP 2024。用激活修补和参数恢复说明，微调式遗忘改变的是知识的提取路径，而不是删除知识。（与本项目一致）
- [Unlearning Sensitive Information in Multimodal LLMs: Benchmark and Attack-Defense Evaluation](https://arxiv.org/abs/2505.01456) — Vaidehi Patil et al.，arXiv 2025。同样在 LLaVA-1.5-7B 上：用 logit lens 读各层隐藏状态的白盒攻击能找回被删除的信息，但只和遗忘前的模型比，没有与从未见过这些信息的模型对照。是本项目实验 1 最接近的已有工作。（与本项目最相关）
- [Extracting Unlearned Information from LLMs with Activation Steering](https://arxiv.org/abs/2411.02631) — Atakan Seyitoğlu et al.，NeurIPS 2024 Workshop（Safe Generative AI）。用激活引导提取已遗忘的信息；对广为人知的知识有效，对非公众人物的细节较弱（本项目的虚构人物正属于后者）。
- [Measuring the Depth of LLM Unlearning via Activation Patching](https://arxiv.org/abs/2605.24614) — Jaeung Lee, Dohyun Kim, Jaemin Jo，EMNLP 2026。以 retain 模型为基线做激活修补，提出 0 到 1 的“遗忘深度”指标，度量目标知识还有多少能被恢复。与本项目以 Oracle 为锚的思路接近。（与本项目最相关）
- [Do LLMs Really Forget? Hidden-State Leakage in Model Unlearning and How to Fix it](https://arxiv.org/abs/2609.36612) — Hadi Reisizadeh et al.，arXiv 2026。输出不再泄漏时，隐藏状态里仍常能读出敏感信息；只看输出会高估遗忘。（与本项目一致）
- [Multilingual Unlearning in LLMs: Transfer, Dynamics, and Reversibility](https://arxiv.org/abs/2606.03291) — Chaoyi Xiang et al.，ICML 2026。遗忘主要是后几层的表层压制，一个推理时的引导方向就能恢复约 50%（Qwen）到 90%（Gemma）的遗忘知识。（与本项目一致）

### 综述、立场与其他代表性工作

- [Rethinking Machine Unlearning for Large Language Models](https://arxiv.org/abs/2402.08787) — Sijia Liu et al.，Nature Machine Intelligence 2025。LLM 遗忘的系统综述：定义、方法与评测。
- [Machine Unlearning Doesn't Do What You Think: Lessons for Generative AI Policy and Research](https://arxiv.org/abs/2412.06966) — A. Feder Cooper et al.，NeurIPS 2025（立场论文）。遗忘的目标与实际能做到的之间存在多处错配，遗忘不是约束生成式 AI 行为的通用方案。
- [MUSE: Machine Unlearning Six-Way Evaluation for Language Models](https://arxiv.org/abs/2407.06460) — Weijia Shi et al.，ICLR 2025。从六个维度评测遗忘；多数方法能压住逐字和知识记忆，但难以避免隐私泄漏。
- [The WMDP Benchmark: Measuring and Reducing Malicious Use With Unlearning](https://arxiv.org/abs/2403.03218) — Nathaniel Li et al.，ICML 2024。危险知识的多选题基准，并提出 RMU 遗忘方法。
- [Large Language Model Unlearning](https://arxiv.org/abs/2310.10683) — Yuanshun Yao, Xiaojun Xu, Yang Liu，NeurIPS 2024。较早系统定义 LLM 遗忘设定和评测的工作之一，只用负样本就能去除不良输出。
- [Who's Harry Potter? Approximate Unlearning in LLMs](https://arxiv.org/abs/2310.02238) — Ronen Eldan, Mark Russinovich，arXiv 2023。约 1 GPU 小时的微调就让 Llama2-7B 无法生成或回忆哈利·波特相关内容。

## 仓库内容

`index.html` 是汇总网页，`assets/` 是网页用到的图。仓库不含实验代码、模型和逐题结果。论文只给出链接，没有转载 PDF。
