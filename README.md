# MMU：多模态大模型遗忘评测（阶段汇总）

在线页面：https://gitdrowner.github.io/MMU/

研究对象是 LLaVA-1.5-7B 在 MLLMU-Bench 上的五种遗忘方法（遗忘 5% 的人物）。要回答的问题是：被"遗忘"的人物知识是真的没了，还是换个条件就能拿出来。所有结论都相对 Oracle 而言。Oracle 从未见过这些人物，但接受了同样的提示或同样的更新。

## 四轮实验

1. **四组重学对照**：在 5 个遗忘人物的数据或无关数据上微调，看另外 20 个遗忘人物的知识会不会回来。
2. **推理路径审计**：让模型先写一段推理再回答，看会不会多暴露知识；推理文本里会不会泄漏身份或属性。
3. **utility 约束下的恢复**：在 retain 准确率下降不超过 2、5、10 个点的条件下，看无关数据上的更新能否让遗忘模型比接受同样更新的 Oracle 暴露更多知识。攻击配置只在 5 个验证人物上选，选定之后才打开 15 个最终测试人物的结果。之后又按主口径（同一个更新对遗忘模型、Full、Oracle 都不超预算）做了重分析，没有新的训练。
4. **IDK 拒答闸门的因果审计**：不做任何训练，在 hidden state 上检查 IDK 的“遗忘”是不是只在知识上加了一个拒答闸门。逐层读出知识，在后层加减一个拒答方向（steering），在模型之间替换各层的 residual（activation patching）。方向、层和强度只在 5 个验证人物上选，判定规则在汇总结果之前写定。

## 目前可以得到的结论

### 比较稳的结论

1. **遗忘方法改变的主要是"用什么方式能读出知识"，而不是把知识删掉。** 除了被训练破坏的 GA-Diff ep10，其余四个遗忘模型都能在某种读法下显著高于 Oracle：IDK 在似然读数下是 0.88（Oracle 0.26）；NPO 和 GA-Diff ep2 加一句"直接回答"后分别高出 +0.21\* 和 +0.09\*；KL-Min 在姓名题上高出 +0.31\*。（第二、三轮一致）
2. **IDK 的遗忘是一个绑定在训练问法上的拒答闸门，而且很容易打开。** 原始问法下 100% 拒答，换一句提示只剩 3%。一次让 IDK、Full、Oracle 的 retain 各只降 3 到 4 个点的无关更新，就把拒答率降到 13%，生成答对率从 0 升到 0.61，同一更新下的 Oracle 只有 0.18。（三轮一致，两种口径都成立）
3. **GA-Diff ep10 不算遗忘成功，而是模型被破坏了。** 十选一准确率 0.07，低于随机水平，也低于 Oracle，生成的全是乱码。无关更新在预算内毫无作用；预算外修复了部分生成能力，但准确率只回到 Oracle 的水平。（三轮一致）
4. **Oracle 不会从同样的提示或更新里获得目标知识，所以"高出 Oracle"可以归为残留知识。** 第二轮 7 种问法、20 个更新检查点下，Oracle 都在 0.21–0.28；第三轮全部 63 个更新后检查点上在 0.24–0.27（更新前 0.265）。（三轮一致）
5. **让模型先写一段推理，不会多暴露知识。** 写完三段 rationale 后没有任何模型的准确率上升（Full −0.24\*，IDK −0.21\*），rationale 里写的身份对属性答案影响很小。在这个模型上，用 CoT 审计遗忘不如似然读数灵敏。（第二轮）
6. **属性知识绑定在人脸图像上，不经由姓名。** 不给图像时所有模型都回到 Oracle 的水平；直接告诉模型正确姓名也只有 0.22–0.33（Oracle 0.26），在训练过的 retain 人物上同样如此。（第二轮）

### 第三轮：按主口径重分析（同一个更新对遗忘模型、Full、Oracle 都不超预算）

7. **只有 IDK 通过 release gate，其余遗忘模型在任何更新之前就已受损。** 以 5 个点为容差，NPO、GA-Diff ep2、KL-Min 的 retain 准确率在更新之前就比 Full 低 13、29、26 个点；GA-Diff ep10 对遗忘题的回答 100% 是乱码，判为 collapsed。
8. **按主口径，只剩 IDK 的拒答闸门坍塌和 NPO 的更新后残留。**（主口径是在最终测试人物解封后才定的，不是盲测确认）
   - IDK：在 5 和 10 个点预算下，同一个更新（lr 1e-4 × 5 步）打开了拒答闸门，见第 2 条。十选一的残留在更新之前就在，由更新带来的部分约为 0。
   - NPO：在 10 个点预算下高出同更新的 Oracle +0.15\*，Holm 校正后仍显著，15 人中 13 人为正。但由更新带来的部分（UID +0.10\*）在 Holm 校正后不显著，所以不能确定是更新造成的。
   - GA-Diff ep2 只剩未校正的 +0.11\*（5 和 10 个点），KL-Min 是 −0.02，都没有检测到恢复。
9. **GA-Diff ep2 和 KL-Min 的恢复只在"只看被攻击模型自己的 utility"时成立。** 按这个补充口径（也是事先登记的盲测分析），两者在 2 个点预算内就高出同更新的 Oracle +0.19\* 和 +0.20\*，由更新带来的部分也在 Holm 校正后显著。但那些更新让 Full 和 Oracle 的 retain 掉 10 到 26 个点；它们自己几乎不掉，是因为起点本来就低。
10. **不存在对所有方法都有效的"通用良性更新"。** 对五个方法都可行的更新基本是空操作；去掉 GA-Diff ep10 后，最好的通用更新能让三个方法同时显著，但它让 Full 和 Oracle 掉 10 到 11 个点。

### 第四轮：IDK 的 hidden state 和因果干预（只分析了 IDK）

按事先写定的规则判为 **B（部分支持）**：六条判据里 K2 和 K5 没有通过。

11. **IDK 在拒答之前就已经算出了答案，而且这部分知识和正确身份绑定。** 逐层读出正确答案比最强干扰项高多少，IDK 从第 17 层左右开始与 Oracle 分开，和 Full 分开的位置相同。中间层带（第 12–21 层）高出 Oracle +0.055\*，15 人中 13 人为正；后层带（第 22–32 层）+0.55\*，15 人全部为正。同一道题换成另一个人的图像，中间层带的优势几乎全部消失。（层带检验是验证性的）
12. **无关更新没有让这些读数上升。** 更新前后，前、中、后三个层带的变化是 +0.001、+0.009、−0.020，十选一准确率变化 −0.006 [−0.049, +0.034]。更新后能答对，是因为闸门打开、原有的知识被放了出来。只能说没有观察到重新学习，区间允许最多约 +0.03 的提升。后层带的 −0.020 超出事先定的 ±0.018，K2 因此不通过；但这是下降，同一更新让 Full 的后层带降了 0.050。
13. **拒答与知识可以分开，并且能被因果地开关。** 在第 30 层减去一个拒答方向，IDK 更新前的拒答率从 1.00 降到 0.65；在第 32 层把它加到更新后的模型上，拒答率从 0.13 升到 0.55。把更新后模型的 residual 换进更新前的模型，闸门打开，知识读数没有显著变化；反方向能把它关上。随机方向几乎没有作用；Oracle 在同样的 steering 和 patching 下既不拒答，也没有多答对。
14. **但它不是一个“后层闸门”。** 把更新前模型第 1–4 层的状态放回更新后的模型，就足以让它重新拒答。反过来，要把更新后的状态一直换到第 13 层，拒答才降一半，换到第 24 层才降到 0.30。只把 Oracle 第 1 层的 residual 换进 IDK，拒答就从 1.00 降到 0.01。后层方向只能部分地调节拒答：这样打开后答对的比例和同一干预下的 Oracle 一样（0.15），而更新放出来的是真知识（0.60）。（K5 因此不通过）
15. **触发信号可能在 IDK 自己的图像 token 里。** 第 1 层只把 Oracle 的图像 token 换进来，拒答就降到 0.01；只换文本位置没有作用。到第 16、24 层，拒答已经写进文本位置。无关更新改的主要是文本一侧。（探索性，看到逐层曲线后才追加，需要在另一个划分上重复）

### 对论文的含义（建议）

- **以 Oracle 为锚的评测是方法上的核心**：同一读法、同一更新也施加在 Oracle 上，才能把残留知识和读法或更新本身的效应分开。
- **utility 约束要对三个模型同时成立，而且要先过 release gate。** 只看被攻击模型自己的 utility，会把对健康模型有破坏性的更新当成良性更新；在 release 时就已受损的模型，它的"恢复"要单独标注。
- **拒答式遗忘（IDK）只在完好的知识上加了一层拒答**：知识在 hidden state 里基本完整，闸门在一个对三个模型都很小的良性更新下就会打开。这个闸门在最早几层就被触发、由很多层写入，应写成 early-triggered, distributed refusal gate，不要写成后层闸门。
- **hidden-state 证据以因果干预为准。** 余弦、CKA 这类原始距离分不开知识和其他训练差异；能区分“知识还在”的，是与身份绑定的中层读数，以及“换入 Oracle 的状态后还能不能答对”。Counterfactual Trajectory Matching 如果要做，建议只做在因果定位到的层和位置上、用知识读出而不是原始距离的窄版本，并先用第二个 Oracle 种子估计噪声底线。
- **UID 的定义需要再考虑。** 现在的 UID 只比 PUR 少一个与更新无关的常数，按 UID 选和按 PUR 选必然是同一个检查点；要让两者能区分，需要一个随更新变化的基线。
- **写成论文主结论之前，至少要在另一个数据划分和第二个 Oracle 种子上复现。**

\* 表示 95% 置信区间不含 0。区间来自按人物聚类的 bootstrap（10,000 次重采样），模型之间的差是配对差。第三轮写"Holm 校正后"的地方，是对 5 个方法 × 3 个预算共 15 个比较做的校正。证据范围：一个模型、一个数据划分、一个随机种子、一个 Oracle 种子；第三轮的最终测试只有 15 个人物，第四轮沿用这 15 个人物，Holm 校正在每一族检验内部做。

## 相关论文

按与本项目的关系分 6 组，共 58 篇。链接指向 arXiv 摘要页，这里不转载论文全文。

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
- [Unlearning Isn't Deletion: Investigating Reversibility of Machine Unlearning in LLMs](https://arxiv.org/abs/2505.16831) — Xiaoyu Xu et al.，ICML 2026。少量微调就能恢复遗忘前的行为；用 PCA、CKA、Fisher 信息等表征指标说明知识被压制而非删除。第四轮也算了 CKA 等表征距离，但只作描述：距离本身分不开知识和其他训练差异。（与本项目最相关）
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

### hidden state 分析：方法基础与最相关的已有工作（第四轮）

- [interpreting GPT: the logit lens](https://www.lesswrong.com/posts/AcKRB8wDpdaN6v6ru/interpreting-gpt-the-logit-lens) — nostalgebraist，LessWrong 博客 2020。用最终层的 unembedding 直接解码中间层，是逐层读出预测的最简单方法。第四轮的逐层知识读数用的就是它。
- [Eliciting Latent Predictions from Transformers with the Tuned Lens](https://arxiv.org/abs/2303.08112) — Nora Belrose et al.，arXiv 2023。为每一层训练一个仿射映射再解码，比 logit lens 更可靠地逐层读出预测。
- [Locating and Editing Factual Associations in GPT](https://arxiv.org/abs/2202.05262) — Kevin Meng et al.，NeurIPS 2022。用因果追踪把事实知识定位到中间层的 MLP。第四轮的逐层 patching 沿用“替换某一层的状态、看输出怎么变”的思路，只是供体换成了另一个模型。
- [Refusal in Language Models Is Mediated by a Single Direction](https://arxiv.org/abs/2406.11717) — Andy Arditi et al.，NeurIPS 2024。拒答由残差流中的单一方向介导，去掉或加上这个方向就能开关拒答。第四轮用同样的均值差方法找到了 IDK 的拒答方向，但它只能部分地开关拒答；patching 显示 IDK 的拒答在第 1 层就被触发，不能归结为一个后层方向。
- [Representation Engineering: A Top-Down Approach to AI Transparency](https://arxiv.org/abs/2310.01405) — Andy Zou et al.，arXiv 2023。从群体层面的表征出发监测和操控高层概念，可用来寻找并操控与遗忘相关的方向。
- [Designing and Interpreting Probes with Control Tasks](https://arxiv.org/abs/1909.03368) — John Hewitt, Percy Liang，EMNLP-IJCNLP 2019。提出控制任务和选择性指标，区分探针读到的是表征里的信息，还是探针自己学会的。
- [Probing Classifiers: Promises, Shortcomings, and Advances](https://arxiv.org/abs/2102.12452) — Yonatan Belinkov，Computational Linguistics 2022。系统评述探针方法的局限和改进，例如对照基线和因果干预。
- [Intrinsic Test of Unlearning Using Parametric Knowledge Traces](https://arxiv.org/abs/2406.11614) — Yihuai Hong et al.，EMNLP 2025。用词表投影找到参数里的概念向量：现有遗忘方法几乎不改动它们，主要是在推理时压制。（与本项目一致）
- [Dissecting Fine-Tuning Unlearning in Large Language Models](https://arxiv.org/abs/2410.06606) — Yihuai Hong et al.，EMNLP 2024。用激活修补和参数恢复说明，微调式遗忘改变的是知识的提取路径，而不是删除知识。（与本项目一致）
- [Unlearning Sensitive Information in Multimodal LLMs: Benchmark and Attack-Defense Evaluation](https://arxiv.org/abs/2505.01456) — Vaidehi Patil et al.，arXiv 2025。同样在 LLaVA-1.5-7B 上：用 logit lens 读各层隐藏状态的白盒攻击能找回被删除的信息，但只和遗忘前的模型比，没有与从未见过这些信息的模型对照。是第四轮逐层读数最接近的已有工作；第四轮加入了从未见过这些人物的 Oracle 作对照。（与本项目最相关）
- [Extracting Unlearned Information from LLMs with Activation Steering](https://arxiv.org/abs/2411.02631) — Atakan Seyitoğlu et al.，NeurIPS 2024 Workshop（Safe Generative AI）。用激活引导提取已遗忘的信息；对广为人知的知识有效，对非公众人物的细节较弱（本项目的虚构人物正属于后者）。
- [Measuring the Depth of LLM Unlearning via Activation Patching](https://arxiv.org/abs/2605.24614) — Jaeung Lee, Dohyun Kim, Jaemin Jo，EMNLP 2026。以 retain 模型为基线做激活修补，提出 0 到 1 的“遗忘深度”指标，度量目标知识还有多少能被恢复。与第四轮把 Oracle 的状态换进 IDK 的做法思路接近。（与本项目最相关）
- [Do LLMs Really Forget? Hidden-State Leakage in Model Unlearning and How to Fix it](https://arxiv.org/abs/2609.36612) — Hadi Reisizadeh et al.，arXiv 2026。输出不再泄漏时，隐藏状态里仍常能读出敏感信息；只看输出会高估遗忘。（与本项目一致）
- [Multilingual Unlearning in LLMs: Transfer, Dynamics, and Reversibility](https://arxiv.org/abs/2606.03291) — Chaoyi Xiang et al.，ICML 2026。遗忘主要是后几层的表层压制，一个推理时的引导方向就能恢复约 50%（Qwen）到 90%（Gemma）的遗忘知识。第四轮在 IDK 上看到的不完全一样：后层方向只打开了约三分之一的拒答，拒答在第 1 层就被触发。（与本项目一致）
- [Similarity of Neural Network Representations Revisited](https://arxiv.org/abs/1905.00414) — Simon Kornblith et al.，ICML 2019。提出用 CKA 比较不同网络、不同层的表征，在表征维度高于样本数时仍然可用。第四轮的轨迹相似性用了 CKA。
- [Towards Best Practices of Activation Patching in Language Models: Metrics and Methods](https://arxiv.org/abs/2309.16042) — Fred Zhang, Neel Nanda，ICLR 2024。系统比较 activation patching 的评价指标和破坏输入的方式，发现这些细节会改变定位结论。第四轮对每次 patching 同时报告拒答率、答对率和知识读数三个指标。
- [How to use and interpret activation patching](https://arxiv.org/abs/2404.15255) — Stefan Heimersheim, Neel Nanda，arXiv 2024。activation patching 的用法和解读指南，包括替换方向不同（把干净运行的状态换进被破坏的运行，或者反过来）时结论的含义也不同。第四轮“打开”和“关上”两个方向的结果不对称，需要分开解读。

### 综述、立场与其他代表性工作

- [Rethinking Machine Unlearning for Large Language Models](https://arxiv.org/abs/2402.08787) — Sijia Liu et al.，Nature Machine Intelligence 2025。LLM 遗忘的系统综述：定义、方法与评测。
- [Machine Unlearning Doesn't Do What You Think: Lessons for Generative AI Policy and Research](https://arxiv.org/abs/2412.06966) — A. Feder Cooper et al.，NeurIPS 2025（立场论文）。遗忘的目标与实际能做到的之间存在多处错配，遗忘不是约束生成式 AI 行为的通用方案。
- [MUSE: Machine Unlearning Six-Way Evaluation for Language Models](https://arxiv.org/abs/2407.06460) — Weijia Shi et al.，ICLR 2025。从六个维度评测遗忘；多数方法能压住逐字和知识记忆，但难以避免隐私泄漏。
- [The WMDP Benchmark: Measuring and Reducing Malicious Use With Unlearning](https://arxiv.org/abs/2403.03218) — Nathaniel Li et al.，ICML 2024。危险知识的多选题基准，并提出 RMU 遗忘方法。
- [Large Language Model Unlearning](https://arxiv.org/abs/2310.10683) — Yuanshun Yao, Xiaojun Xu, Yang Liu，NeurIPS 2024。较早系统定义 LLM 遗忘设定和评测的工作之一，只用负样本就能去除不良输出。
- [Who's Harry Potter? Approximate Unlearning in LLMs](https://arxiv.org/abs/2310.02238) — Ronen Eldan, Mark Russinovich，arXiv 2023。约 1 GPU 小时的微调就让 Llama2-7B 无法生成或回忆哈利·波特相关内容。

## 仓库内容

`index.html` 是汇总网页，`assets/` 是网页用到的图。仓库不含实验代码、模型和逐题结果。论文只给出链接，没有转载 PDF。
