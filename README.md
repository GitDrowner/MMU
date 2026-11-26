# MMU：多模态大模型遗忘评测（阶段汇总）

在线页面：https://gitdrowner.github.io/MMU/

第九轮人工复核的标注工具：[`annotate/index.html`](annotate/index.html)（在线地址 https://gitdrowner.github.io/MMU/annotate/ ；用法见本文末尾“标注工具”一节）

研究对象是 LLaVA-1.5-7B 在 MLLMU-Bench 上的五种遗忘方法（遗忘 5% 的人物）。要回答的问题是：被"遗忘"的人物知识是真的没了，还是换个条件就能拿出来。所有结论都相对 Oracle 而言。Oracle 从未见过这些人物，但接受了同样的提示或同样的更新。第六轮训练了第三个 Oracle 种子，并在第二种架构 Qwen2.5-VL-7B 上复现 IDK 审计的最精简版本。第七轮用不含答案模板的读数复核另外四种方法的 hidden-state 结论，并为 Qwen 训练了第二个 Oracle 种子。第八轮用受控的离线干预检验整句平均读数的伪影，核验评分的数值精度并把评分改为 fp32，然后在另外 50 个人物上为 NPO 做确认。第九轮没有新的训练：审计同一个无关小更新对自由生成的代价，检验把生成效用放进约束之后释放是否还在预算内，并固定论文的主结果和表述。第十轮沿 10 个强度不同的无关更新画出“释放对代价”的曲线，并对 IDK 的开闸更新做生成效用审计。第十一轮为 forget_10 训练第二个 Oracle 种子和第二个 NPO 训练种子，复核前三轮在 forget_10 上的结论。

## 十一轮实验

1. **四组重学对照**：在 5 个遗忘人物的数据或无关数据上微调，看另外 20 个遗忘人物的知识会不会回来。
2. **推理路径审计**：让模型先写一段推理再回答，看会不会多暴露知识；推理文本里会不会泄漏身份或属性。
3. **utility 约束下的恢复**：在 retain 准确率下降不超过 2、5、10 个点的条件下，看无关数据上的更新能否让遗忘模型比接受同样更新的 Oracle 暴露更多知识。攻击配置只在 5 个验证人物上选，选定之后才打开 15 个最终测试人物的结果。之后又按主口径（同一个更新对遗忘模型、Full、Oracle 都不超预算）做了重分析，没有新的训练。
4. **IDK 拒答闸门的因果审计**：不做任何训练，在 hidden state 上检查 IDK 的“遗忘”是不是只在知识上加了一个拒答闸门。逐层读出知识，在后层加减一个拒答方向（steering），在模型之间替换各层的 residual（activation patching）。方向、层和强度只在 5 个验证人物上选，判定规则在汇总结果之前写定。之后用同一套逐层读数补做了另外四种方法，包括在另外 10 个人物上的重复检验和对照。
5. **复现、第二个 Oracle 种子、三种攻击**：在另外 50 个人物（forget_10）上重新训练 IDK 和 Oracle，复现第四轮，并检验事先登记的“图像触发 → 文本传播”；训练第二个 Oracle 种子，估计 Oracle 轨迹的噪声底线，并以它为锚重算第三轮；比较普通无关更新、只在第 13–24 层文本位置训练的 LoRA（TRACEBREAK-MM）和后层 steering 三种攻击。三份判定规则都在产生相关输出之前写定。
6. **第三个 Oracle 种子、Qwen2.5-VL 上的最精简 IDK 审计**：训练第三个 Oracle 种子，用三个种子估计 Oracle 锚点的噪声底线，并按事先写定的规则改以三个种子的平均 Oracle 作锚点；在 Qwen2.5-VL-7B 上按 LLaVA 的配方训练 Full、Oracle 和 IDK，用同样的题、人物和更新网格检验 IDK 的四个核心发现。两份判定规则都在相应模型训练之前写定。
7. **另外四种方法的去模板化复核、第二个 Qwen Oracle 种子**：对 9 个 LLaVA 模型逐个 token 重新打分，用去掉所有候选共有前缀的读数和只看属性值的读数，复核第四轮补充里另外四种方法的结论（窗口固定在第 20–28 层，15 个发现人物加另外 10 人复现，另做两种改写的答案模板）；为 Qwen2.5-VL-7B 训练第二个 Oracle 种子，施加同样的 63 个更新，按四种可行性规则报告，并在开闸检查点补做逐层读数。规则在任何 GPU 输出之前写定，是带时间戳的内部预登记。
8. **评分伪影的受控实验、fp32 评分核验、forget_10 上的 NPO 确认**：A，在已经算好的逐 token 分数上只改所有候选共有的前缀项（候选其余部分的分数、长度和标签逐字节不变），看整句平均读数的准确率、排序和相对 Oracle 的结论怎么变；B，把评分拆成几级（fp16 分块、fp32 的 lens、全部候选同批、fp32 权重和前向）逐级比较，查清第七轮数值偏差的来源，并用 fp32 对完整人物集重新打分、重跑第七轮的聚合；C，在另外 50 个人物（forget_10）上按原配方训练 NPO，主 checkpoint 取固定预算的第 10 轮，测试人物的输出在评分核验通过、release 效用算完之后才打开，检验只运行一次，之后才做同一个无关小更新。规则在运行前写定，是带时间戳的内部预登记。
9. **生成效用审计、带生成约束的释放、论文结果冻结**：没有任何训练，只评测第八轮冻结的那一个更新。把 retain 人物的自由生成从 60 题补齐到 236 题（原程序，旧的 60 条回答逐字复现），用事先冻结的自动标签判每条回答是否披露正确值、是否含矛盾值、是否拒答、是否无效；比较 Full、Oracle、NPO 在更新前后的 retain 识别和自由生成；在原效用规则之外加上“retain 自由生成降幅不超过 2、5、10 个点”的约束，看更新后的生成释放是否还在预算内；为人工复核准备盲化的分层抽样包和标注工具；把论文的主结果、表述和统计口径固定成三个文件。规则在运行前写定，是带时间戳的内部预登记；40 个测试人物以前分析过，不是盲测。
10. **更新强度曲线上的生成效用、IDK 开闸更新的生成审计**：
    - **没有新训练。** 没有训练遗忘模型或 Oracle，只给 NPO 补跑 lr 5e-5 的同一个无关更新。
    - **评测。** 在固定的网格（两个学习率 × 1、2、5、10、20 步）上，评测 NPO、IDK、Full、Oracle 对遗忘人物的披露，以及 retain 识别和自由生成。
    - **选点。** 只用 10 个验证人物为 2、5、10 个点预算选更新，选点写入只读文件之后才打开 40 个测试人物。
    - **forget-5。** 对第三轮和第六轮为 IDK 选出的两个开闸更新补做 retain 自由生成，以三个 Oracle 种子为锚检验生成预算。
    - **规则。** 在运行前写定，是带时间戳的内部预登记；人物以前分析过，不是盲测。
11. **forget_10 上的第二个 Oracle 种子和第二个 NPO 训练种子**：
    - **训练。** 按原配方只换种子，训练 Oracle 和 NPO。
    - **更新和读数。** 施加第十轮的全部更新，用 fp32 评分读第 20–28 层。
    - **复核。** 以种子 42、种子 43 和两者的平均三个锚点，复核第八到第十轮在 forget_10 上的 9 项结论；两个 NPO 种子并列报告。
    - **规则。** 在训练前写定；第二个 NPO 的 release 判定在打开测试人物之前写定。

## 目前可以得到的结论

### 比较稳的结论

1. **遗忘方法改变的主要是"用什么方式能读出知识"，而不是把知识删掉。** 除了被训练破坏的 GA-Diff ep10，其余四个遗忘模型都能在某种读法下显著高于 Oracle：IDK 在似然读数下是 0.88（Oracle 0.26）；NPO 和 GA-Diff ep2 加一句"直接回答"后分别高出 +0.21\* 和 +0.09\*；KL-Min 在姓名题上高出 +0.31\*。（第二、三轮一致）
2. **IDK 的遗忘是一个绑定在训练问法上的拒答闸门，而且很容易打开。** 原始问法下 100% 拒答，换一句提示只剩 3%。一次让 IDK、Full、Oracle 的 retain 各只降 3 到 4 个点的无关更新，就把拒答率降到 13%，生成答对率从 0 升到 0.61，同一更新下的 Oracle 只有 0.18。同一更新让第二、第三个 Oracle 种子的 retain 降 5.4 和 6.6 个点；在另外 50 个人物上，更新后答对率同样高出 Oracle +0.50\*；在 Qwen2.5-VL-7B 上，同一更新把 IDK 的拒答降到 33%（第 23 条）。第十轮补充：放到生成预算下，forget-5 的两个开闸更新都不满足 5 个点；forget_10 上 5 个点的匹配预算内没有检测到释放（第 38、39 条）。（LLaVA 上五轮一致，Qwen 上重复；5 个点预算下主口径是否可行取决于锚点）
3. **GA-Diff ep10 不算遗忘成功，而是模型被破坏了。** 十选一准确率 0.07，低于随机水平，也低于 Oracle，生成的全是乱码。无关更新在预算内毫无作用；预算外修复了部分生成能力，但准确率只回到 Oracle 的水平。（三轮一致）
4. **Oracle 不会从同样的提示或更新里获得目标知识，所以"高出 Oracle"可以归为残留知识。** 第二轮 7 种问法、20 个更新检查点下，Oracle 都在 0.21–0.28；第三轮全部 63 个更新后检查点上在 0.24–0.27（更新前 0.265）。（三轮一致）
5. **让模型先写一段推理，不会多暴露知识。** 写完三段 rationale 后没有任何模型的准确率上升（Full −0.24\*，IDK −0.21\*），rationale 里写的身份对属性答案影响很小。在这个模型上，用 CoT 审计遗忘不如似然读数灵敏。（第二轮）
6. **属性知识绑定在人脸图像上，不经由姓名。** 不给图像时所有模型都回到 Oracle 的水平；直接告诉模型正确姓名也只有 0.22–0.33（Oracle 0.26），在训练过的 retain 人物上同样如此。（第二轮）

### 第三轮：按主口径重分析（同一个更新对遗忘模型、Full、Oracle 都不超预算）

7. **只有 IDK 通过 release gate，其余遗忘模型在任何更新之前就已受损。** 以 5 个点为容差，NPO、GA-Diff ep2、KL-Min 的 retain 准确率在更新之前就比 Full 低 13、29、26 个点；GA-Diff ep10 对遗忘题的回答 100% 是乱码，判为 collapsed。第八轮补充：这是 forget-5 上的结果；在另外 50 个人物（forget_10）上按同一配方训练的 NPO 通过了 release gate，retain 只比 Full 低 1.4 个点（第 31 条）。第九轮补充：这个判定要带限定，见第 35 条。
8. **按主口径，只剩 IDK 的拒答闸门坍塌和 NPO 的更新后残留。**（主口径是在最终测试人物解封后才定的，不是盲测确认）
   - IDK：在 5 和 10 个点预算下，同一个更新（lr 1e-4 × 5 步）打开了拒答闸门，见第 2 条。以第二、第三个 Oracle 种子或三个种子的平均为锚时，5 个点预算下这个更新不可行，10 个点下仍成立（第 20、22 条）。十选一的残留在更新之前就在，由更新带来的部分约为 0。
   - NPO：在 10 个点预算下高出同更新的 Oracle +0.15\*，Holm 校正后仍显著，15 人中 13 人为正。但由更新带来的部分（UID +0.10\*）在 Holm 校正后不显著，所以不能确定是更新造成的。
   - GA-Diff ep2 只剩未校正的 +0.11\*（5 和 10 个点），KL-Min 是 −0.02，都没有检测到恢复。
9. **GA-Diff ep2 和 KL-Min 的恢复只在"只看被攻击模型自己的 utility"时成立。** 按这个补充口径（也是事先登记的盲测分析），两者在 2 个点预算内就高出同更新的 Oracle +0.19\* 和 +0.20\*，由更新带来的部分也在 Holm 校正后显著。但那些更新让 Full 和 Oracle 的 retain 掉 10 到 26 个点；它们自己几乎不掉，是因为起点本来就低。
10. **不存在对所有方法都有效的"通用良性更新"。** 对五个方法都可行的更新基本是空操作；去掉 GA-Diff ep10 后，最好的通用更新能让三个方法同时显著，但它让 Full 和 Oracle 掉 10 到 11 个点。

### 第四轮：IDK 的 hidden state 和因果干预

按事先写定的规则判为 **B（部分支持）**：s0 上六条判据里 K2 和 K5 没有通过。第五轮在另外 50 个人物上复现，仍为 B，只有 K5 没有通过。

11. **IDK 在拒答之前就已经算出了答案，而且这部分知识和正确身份绑定。** 逐层读出正确答案比最强干扰项高多少，IDK 从第 17 层左右开始与 Oracle 分开，和 Full 分开的位置相同。中间层带（第 12–21 层）高出 Oracle +0.055\*，15 人中 13 人为正；后层带（第 22–32 层）+0.55\*，15 人全部为正。同一道题换成另一个人的图像，中间层带的优势几乎全部消失。在另外 50 个人物上，中间层带是 +0.065\*，40 人中 38 人为正。
12. **无关更新没有让这些读数上升。** 更新前后，前、中、后三个层带的变化是 +0.001、+0.009、−0.020，十选一准确率变化 −0.006 [−0.049, +0.034]；在另外 50 个人物上是 −0.003、+0.002、−0.017，十选一 −0.003 [−0.033, +0.027]。更新后能答对，是因为闸门打开、原有的知识被放了出来。只能说没有观察到重新学习。K2 在 s0 上以 0.002 之差不通过，在 forget_10 上通过。
13. **拒答与知识可以分开，并且能被因果地开关。** 在第 30 层减去一个拒答方向，IDK 更新前的拒答率从 1.00 降到 0.65；在第 32 层把它加到更新后的模型上，拒答率从 0.13 升到 0.55。把更新后模型的 residual 换进更新前的模型，闸门打开，知识读数没有显著变化；反方向能把它关上。随机方向几乎没有作用；Oracle 在同样的 steering 和 patching 下既不拒答，也没有多答对。在另外 50 个人物上，验证人物选出的层和强度与 s0 相同，拒答分别从 1.00 降到 0.52、从 0.09 升到 0.34。
14. **但它不是一个“后层闸门”。** 把更新前模型第 1–4 层的状态放回更新后的模型，就足以让它重新拒答。反过来，要把更新后的状态一直换到第 13 层，拒答才降一半，换到第 24 层才降到 0.30。只把 Oracle 第 1 层的 residual 换进 IDK，拒答就从 1.00 降到 0.01。在另外 50 个人物上，打开和关上闸门的 50% 点同样在第 13 层和第 2 层。后层方向只能部分地调节拒答：这样打开后答对的比例和同一干预下的 Oracle 一样（0.15；在另外 50 个人物上只多 0.06），而更新放出来的是真知识（0.60）。（K5 在两组人物上都不通过）
15. **拒答的决定从中层起由文本位置承载，无关更新改的是文本一侧，身份知识在 IDK 自己的图像 token 上；但触发信号不只在图像 token 里。** 第 16、24 层只换入 Oracle 的文本位置，拒答降到 0；第 24 层只换图像 token 毫无作用。把更新后的图像 token 换进更新前的模型，任何层都不改变拒答，只换文本位置才行。这些在 s0 上是探索性发现，在另外 50 个人物上按事先登记的预测全部重复。s0 上第 1 层只换 Oracle 的图像 token 就能打开闸门、只换文本位置没有作用；但在另外 50 个人物上，只换第 1 层的文本位置也能打开（拒答 1.00 → 0.18），所以事先登记的“图像触发 → 文本传播”作为整体不成立。

### 第四轮补充：另外四种方法的 hidden state（s0，规则都在运行前写定）

16. **NPO 和 GA-Diff ep2 在中后层保有与身份绑定、高于 Oracle 的读数，但到输出层都消失。** 按事先规则，只有 NPO 在更新前的中间层带显著高于 Oracle（+0.037，Holm 0.029）；KL-Min、GA-Diff ep10 都没有检测到。第 20–28 层的优势是事后看到的，事先固定这个窗口后，在另外 10 个人物上对 NPO（+0.118）和 GA-Diff ep2（+0.090）重复成立，并且换成别人的图像就消失。同一个小更新抬高了 NPO、GA-Diff ep2、KL-Min 的后层读数（+0.08 到 +0.19），Oracle 不变。（第七轮部分更正：用不含模板的读数复核，NPO 的结论成立；GA-Diff ep2 的身份绑定只在以种子 42 的 Oracle 为锚时重复，“已重复”撤回；在输出层消失的主要是答案开头，见第 25、26 条。）
17. **挡住 NPO 输出的主要是对答案开头的压制，这个压制与身份无关。** 每道题的所有候选句共用第一个词元。NPO 在遗忘人物上把它的对数概率压到 −35（Oracle −0.3），压制在最后 3 层形成，第三轮的更新把它去掉。把 NPO 前几层的状态换成 Oracle 的，输出读数大幅上升，但换成另一个遗忘人物的图像也一样（+0.891 对 +0.894）。与身份绑定的部分只在不受开头影响的十选一（按总和打分）上看得到：比 Oracle 多 +0.11，只在正确图像下出现。（压制是事后诊断出来的）
18. **NPO 的优势由它自己的中后层算出，压制也不在最后几层的权重里。** 把 Oracle 前 11 层的状态换进 NPO，第 28 层的优势基本不变；把 NPO 或 GA-Diff ep2 最后 2 层或 4 层的权重换成 Oracle 的，事先登记的三项预测都不成立。

这些逐层读数是整句平均的 margin，所有候选共用的答案开头会混进来；IDK 的结论在第五轮用不含模板的读数复核过，另外四种方法在第七轮复核（第 25–27 条）。

### 第五轮：复现、第二个 Oracle 种子、三种攻击（规则都在运行前写定）

19. **IDK 的第四轮结论在另外 50 个人物上重复。** 在 forget_10（与之前的人物只重合 1 人）上重新训练 IDK 和 Oracle，用同一个更新，在 40 个测试人物上按第四轮的规则原样检验：分类同为 B，事先登记的五条预测全部成立。同一更新后，IDK 的答对率高出同更新的 Oracle +0.50\*（s0 是 +0.42\*）。用不含模板的读数，K1、K2、K6 的结论不变。
20. **换一个 Oracle 种子，主要结论不变，卡在边界上的会变。** 第二个 Oracle（种子 43）与原 Oracle 在遗忘人物上的十选一只差 −0.025（不显著）；同一更新下两条 Oracle 轨迹之差，中位数 0.006，90 分位 0.024，最大 0.063。IDK 的更新前残留（+0.59 → +0.63）、IDK 在 10 个点预算下的闸门坍塌（+0.43 → +0.47）、NPO 在 10 个点预算下的更新后残留（+0.15 → +0.18）都不变。变了的有：5 个点预算下 IDK 的闸门坍塌变得不可行（同一更新让新 Oracle 的 retain 降 5.4 个点）；NPO 和 GA-Diff ep2 的更新前残留在新种子下变成 Holm 校正后显著（+0.10、+0.12）。45 行结果里 23 行不随种子改变。第六轮加了第三个种子，见第 22 条。
21. **三种攻击里，普通无关更新最有效。** 在 5 个点预算下（主口径），只有普通更新打开了 IDK 的闸门：拒答 1.00 → 0.13，答对率高出同攻击的 Oracle +0.43\*。只在第 13–24 层的文本位置训练 LoRA（TRACEBREAK-MM），数据和步数网格相同，要 lr 1e-4 训练 20 步以上才能打开闸门，那时 Full 和 Oracle 的 retain 已降 6 到 8.5 个点；后层拒答方向 steering 在预算内也打不开。预算放宽到 10 个点时，TRACEBREAK-MM 高出 Oracle +0.27\*，steering +0.09（不显著）。

### 第六轮：第三个 Oracle 种子、Qwen2.5-VL 上的最精简 IDK 审计（规则都在运行前写定）

22. **三个 Oracle 种子：远离边界的结论不变，卡在边界上的要同时报告四个锚点。** 三个 Oracle 在遗忘人物上的十选一是 0.265、0.240、0.243；同一更新下 Oracle 轨迹之差的 90 分位从两个种子时的 0.024 升到 0.034。按事先写定的规则，论文改以三个种子的平均 Oracle 作锚点。第三轮主口径的 45 条结论里，16 条在三个单种子锚点和平均锚点下类别都不变，可以直接陈述，包括 IDK 在 10 个点预算下的闸门坍塌（平均锚点 +0.437\*）、NPO 在 10 个点预算下的更新后残留（+0.149\*，由更新带来的部分仍不显著）、GA-Diff ep10 低于 Oracle。其余 29 条要同时报告四个锚点，包括 5 个点预算下 IDK 的闸门坍塌（第三轮的更新只在原 Oracle 下可行）和多数 2、5 个点预算下的结论，因为种子 44 对更新最敏感。更新前残留里，IDK（平均锚点 +0.605\*）、GA-Diff ep10（−0.202\*）和 KL-Min（不显著）在四个锚点下类别不变；NPO 和 GA-Diff ep2 只在种子 43 下显著，平均锚点下是 +0.074 和 +0.089，不显著。（事先登记的报告规则）
23. **在 Qwen2.5-VL-7B 上，IDK 同样只是在基本完好的知识上加了一道拒答闸门，同一个无关小更新也能打开它。** 按 LLaVA 的配方训练 Full、Oracle 和 IDK，用同样的题、同样的 15 个测试人物和同样的更新网格。IDK 对遗忘人物 100% 拒答，十选一仍有 0.93（Oracle 0.29，Full 0.98），高出 Oracle +0.641\*，15 人全部为正（LLaVA +0.587\*）。LLaVA 第三轮的那个更新（lr 1e-4 × 5 步）把拒答降到 0.33，答对率高出同更新的 Oracle +0.391\*（只约束 IDK 自身 utility 时；LLaVA 是拒答 0.13、+0.428\*）。主口径 10 个点预算下，更温和的 lr 5e-5 × 10 步把拒答降到 0.60，高出 Oracle +0.210\*。（Q1 预注册成立；开闸来自同族检验和补充口径）
24. **但按事先规则只算“部分复现”：中间层带读不出残留，5 个点主口径下也找不到可行的开闸更新。** 按相对深度换算的中间层带（第 10–18 层）IDK 减 Oracle 是 −0.009，换成别人的图像也没有差别；同一族检验里的后层带（第 19–28 层）是 +0.369\*。事后看，残留和身份绑定都从第 21 层（共 28 层）起才持续显著，后层带的身份绑定 +0.324\*，15 人全部为正；而 Oracle 对自己学过的人物，用同一读法也要到第 22 层才读得出（LLaVA 是 32 层中的第 19 层）。所以 LLaVA 的层带位置不能按相对深度搬到 Qwen 上。5 个点预算下选中的更新不开闸（拒答仍为 1.00）：所有能开闸的更新都让 Oracle 的 retain 降 8.1 个点以上，LLaVA 的那个更新让它降 14.3 个点，而同一更新下 IDK 和 Full 只降 3.4 和 2.7 个点。Q4 按规则机械地不成立：它的阈值取中间层带估计的三分之一，估计不大于 0 时直接判为不成立。第七轮补充：“降 8.1 个点以上”只对 Qwen 的 Oracle 种子 42 成立，第二个种子在 lr 5e-5 × 10 步只降 1.7 个点（第 28 条）；预注册判定不变。（预注册：Q1 成立，Q2–Q4 不成立；逐层分析是事后的）

### 第七轮：另外四种方法的去模板化复核、第二个 Qwen Oracle 种子（规则都在运行前写定）

25. **去掉所有候选共有的答案模板后，NPO 在第 20–28 层仍高于 Oracle 并与身份绑定，在另外 10 人上也成立。** 只算共有前缀之后的 token，NPO 高出三个 Oracle 种子的平均 +0.42\*（15 个发现人物）和 +0.25\*（另外 10 人），身份绑定 +0.31\* 和 +0.19\*；三个种子分别作锚、只看属性值的 token、换一种改写的答案模板，都成立。GA-Diff ep2 高出 Oracle 在两组人物上都成立，但身份绑定只在发现人物上成立；KL-Min 只在发现人物上成立，并且集中在正确答案比干扰项长的题目；GA-Diff ep10 的身份绑定是负的，不算残留。（事先登记，NPO 已复现；窗口是第四轮事后发现的）
26. **输出层的“消失”主要是答案开头被压低，不是属性值读不出来。** 四种方法在输出层都把所有候选共有的答案开头压得很低，相对 Oracle：NPO −75、GA-Diff ep2 −15、KL-Min −24、GA-Diff ep10 −1459 nats。给定开头之后，NPO 在输出层读出属性值的能力与 Oracle 无法区分（+0.19，区间含 0，三个种子作锚都如此）。第八轮补充：这是 forget-5 上的结果；在 forget_10 上，NPO 输出层的属性值读数显著高于 Oracle（+0.80\*，第 31 条），“与 Oracle 持平”不能当作 NPO 的一般性质。它在整句平均读数下的输出层亏损随候选长度变号（正确答案更短 −1.78\*，相近 −0.25，更长 +0.84\*），是被压低的模板 token 与长度一起造成的。KL-Min 和 GA-Diff ep10 在输出层的属性值读数确实低于 Oracle。（事先登记；长度分组是事后的）
27. **同一个无关小更新改的主要是输出通道。** lr 1e-4 × 5 步撤掉了 NPO 的开头压制（相对 Oracle −75 → −0.7），并让 NPO、GA-Diff ep2、KL-Min 在输出层的属性值可读性超过同更新的 Oracle（+0.86\*、+0.66\*、+1.92\*，Holm 校正后显著）。NPO 第 20–28 层的读数不变（+0.02）；GA-Diff ep2 和 KL-Min 在这个窗口只小幅上升（+0.13\*、+0.16\*）。第八轮补充：“不变”应写成“未检测到变化”；在 forget_10 上，同一更新让 NPO 这个窗口的读数小幅显著上升（+0.09\*，第 32 条）。（事先登记；第八轮收窄）
28. **Qwen 的第二个 Oracle 种子对更新没有第一个那么脆弱，但两个 Oracle 都不超预算时，仍只在 10 个点预算下开闸。** 两个 Oracle 在遗忘人物上的十选一几乎相同（0.270、0.281）。同一更新下，凡是有明显下降的检查点，种子 43 都比种子 42 降得少（20 个里 20 个），但闸门大部分打开后仍降 10–30 个点，多于 IDK 和 Full。要求 IDK、Full 和两个 Oracle 都不超预算时，2 和 5 个点下选中的更新不开闸，10 个点下选中 lr 5e-5 × 10 步，拒答 1.00 → 0.60，GEN 随锚点在 +0.17\* 到 +0.21\* 之间；只以种子 43 或两个种子的平均作约束时，这个更新在 5 个点下也可行。开闸后 IDK 后层带的残留没有上升（−0.03），在事先定的等价界之内。（事先登记；5 个点下是否可行取决于用哪个 Oracle 作约束）

### 第八轮：评分伪影的受控实验、fp32 评分核验、forget_10 上的 NPO 确认（规则都在运行前写定）

29. **候选特有的证据逐字节不变时，只改所有候选共有的前缀项，就能改变整句平均读数相对 Oracle 的结论。** 离线地把共有答案开头的对数似然换成另一个值，其余 token 的分数、候选长度和标签都不动。NPO 在输出层的整句平均读数，用它自己的前缀项（约 −75 nats）时显著低于 Oracle（−0.63\*）；移除前缀项或换成 Oracle 的，差别就检测不到（−0.07）。同一小更新后的 NPO 显著高于 Oracle（+0.42\*），只给它的前缀项再减 128 nats，就变成显著低于（−0.99\*）。翻转只发生在长度不同的候选之间，与闭式临界点完全一致（每个条件 10,620 对），等长的候选没有一次翻转。整句求和的排序、去前缀读数和只看属性值的读数逐字节不变。（事先登记，不变性是精确的单元测试；这是评分器干预，不是模型干预）
30. **fp16 的评分没有通过一项事先登记的逐 token 核验；改用 fp32 权重和前向之后，第七轮的结论全部不变。** 逐候选单独前向与 12 个一批前向相比，670 万个 token 里有 53 个超出容差，最大差 0.27 nat。原因是 fp16 前向在不同 batch 组成下的舍入噪声，不是实现错误；容差没有放宽。fp32 评分在同一规则下通过（最大 0.0008）。用它对 9 个模型、236 道题重新打分，原样重跑第七轮的聚合：33 项检验的显著性状态无一改变，四种方法的分类不变，每个估计的移动不超过其区间半宽的 2.2%。第七轮那个“求和与去前缀求和的 margin 不相等”的偏差来自分块前向：同一批内共有前缀的值逐位相同，不同批之间最多差 0.16 nat，全部候选同批打分时严格相等。（一项事先登记的核验未通过，评分器已更换；影响评估覆盖完整人物集）
31. **在另外 50 个人物（forget_10）上，NPO 的第 20–28 层同样高于 Oracle 并与身份绑定，而且这个 checkpoint 通过了 release gate。** 按 forget-5 的配方原样训练，取固定预算的第 10 轮：retain 十选一 0.967（Full 0.981），GQA 0.573（0.580），在 2、5、10 个点的容差下都通过；第 2 轮的中间 checkpoint 受损，第 5 轮在 2 个点容差下受损。40 个测试人物上，只看属性值的读数高出 Oracle +0.61\*，身份绑定 +0.49\*，Holm 校正后显著，40 人中 40 人和 38 人为正；正对照 Full 是 +1.01\*，IDK 是 +0.88\*。换读数、分姓名题和属性题、验证人物、旧评分器，判定都相同。输出层的属性值读数也高于 Oracle（+0.80\*）。更新前，它的自由生成答对率 0.07，低于 Oracle 的 0.15，18% 的回答是乱码：残留没有表现为自由生成泄漏。第九轮补充：18% 是旧退化规则的数字，按第九轮的无效输出标签是 99%；它的 retain 自由生成比 Full 低 3.8 个点。“通过 release gate”要带上这两点（第 35 条）。第十一轮补充：换第二个 Oracle 种子、第二个 NPO 训练种子，判定都不变（第 41 条）。（事先登记，测试人物只运行一次；这些人物在第五轮用过；第九轮加限定；第十一轮：两个 Oracle 种子、两个训练种子）
32. **同一个无关小更新解除了 NPO 在 forget_10 上的答案开头压制，之后它的十选一和自由生成都明显高于同更新的 Oracle。** 更新前，NPO 在输出层把共有的答案开头压低 47 nats（相对 Oracle），换成别人的图像也一样；lr 1e-4 × 5 步之后只剩 2 nats。这次更新的数据、batch 顺序、token 数和优化配置与 Full、Oracle 的逐步相同，NPO、Full、Oracle 的 retain 各降 2.7、2.9、4.0 个点：2 个点预算下不可行，5 和 10 个点下两种口径都可行。第九轮补充：这是十选一识别的降幅；同一更新让三个模型的 retain 自由生成各降 7–9 个点，5 个点的生成预算不成立（第 34 条）。更新后 NPO 的十选一 0.72（Oracle 0.28），生成答对 0.46（Oracle 0.14），40 个测试人物上高出同更新的 Oracle +0.46\* 和 +0.34\*。第 20–28 层的属性值读数小幅上升（+0.09\*），输出层 +0.78\*。（事先登记；更新在更新前的检验完成之后才运行；第九轮收窄）

### 第九轮：同一更新的生成效用、带生成约束的释放、论文结果冻结（规则在运行前写定；没有新训练）

33. **同一个无关小更新之后，NPO 在自由生成里说出遗忘人物正确属性的比例明显高于同更新的 Oracle。** 用这一轮事先冻结的回答标签，在 forget_10 的 40 个测试人物上：更新后 NPO 披露正确值的比例是 0.47，Oracle 是 0.14，相差 +0.33\*（35 人为正，Holm 校正后显著）；要求“披露且不含矛盾值”时数值相同。更新前相差 −0.09\*，那时 NPO 的回答几乎全是乱码；更新使这个差增加 +0.42\*。标签是自动规则给出的，人工复核还没做；“更新前低于 Oracle”对乱码回答的判读敏感，只是暂定（第 36 条）。第十一轮补充：“更新前低于 Oracle”只在 NPO 的种子 42 上成立，第二个训练种子更新前高出 +0.07\*；更新后的超出在两个训练种子上都成立（+0.34\*、+0.21\*，以两个 Oracle 种子的平均为锚；第 42 条）。（事先登记，测试人物只聚合一次；这些人物在第五、八轮用过，单个 Oracle 种子；自动标签，人工复核待做）
34. **这次更新不是无损的：十选一识别只降 3–4 个点，自由生成却降 7–9 个点，三个模型都如此。** 这一轮把 retain 人物的自由生成从 60 题补齐到 236 题（25 人）。同一更新后，“披露且不含矛盾值”的比例：NPO 0.911 → 0.831（降 8.0 个点，区间 3.5–12.6），Full 0.949 → 0.878（7.1，3.7–10.7），Oracle 0.902 → 0.817（8.5，5.3–11.5）；十选一同时只降 2.8、3.0、4.1 个点，GQA 不降。丢掉的回答变成了流畅的错误回答（日期差一年、薪水换一个数），乱码率和拒答率都是 0，原来的输出有效性指标看不到。三个模型的降幅之间没有可检出的差别。把生成约束加进去：2 和 5 个点预算都不可行，所有标签判法一致；10 个点只在点估计上可行，三个降幅的 95% 上界都超过 10 个点。按事先写好的措辞，只能说“识别效用保持时目标信息被释放，但 retain 生成有不可忽略的损失”。第十轮补充：这只对这一个更新成立。更弱的更新满足 5 个点的生成预算，在那里 NPO 的披露率仍高于 Oracle，但多出来的部分主要夹在被判为无效的回答里（第 37、40 条）。（事先登记；只测了一个更新；第十轮补了曲线）
35. **旧的退化规则漏掉了大部分乱码，“通过 release gate”要加限定。** 按这一轮的无效输出标签（空白、其他文字、代码片段、大量词表外的词），NPO 更新前对 forget_10 测试人物的回答 99.2% 是无效输出；第三轮沿用至今的退化规则只标出 17.5%。原 release gate 的判定不追溯改写，但它“未 collapsed”的结论依赖这条旧规则：gate 自己规定，遗忘题的回答一半以上是乱码就判 collapsed。另外，NPO 在更新之前的 retain 自由生成比 Full 低 3.8 个点（区间 1.6–6.1）；从未见过遗忘人物的 Oracle 也比 Full 低 4.7 个点，两者持平。gate 只看十选一（差 1.4 个点），看不到这个差距。第十一轮补充：99% 是 NPO 种子 42 的特点，第二个训练种子是 48%，它同样通过 release gate（第 42 条）。（标签事先冻结；辅助阅读抽到的 19 条全部判为无效，但那不是人工复核）
36. **论文的主结果、表述和统计口径已经固定；人工复核还没做。** 主结果表 97 行，每个数字由脚本从源文件按行读出；claim ledger 对六条结论写明了准确表述、适用范围、替代解释、不能说什么、能否进摘要。两处口径没有统一，已逐行标出：第三到第七轮按题加权，第八、九轮先按人物平均；部分读数仍是 fp16 评分。人工复核包有 313 条盲化回答，两位标注者的标注尚未进行，标注工具在本仓库的 `annotate/` 目录。一个只拿到盲化表的语言模型实例读过同一批回答，这不是人工复核：它不改变更新后的超出和“5 个点不可行”；但 NPO 更新前有 438 条“无效且未披露”的回答，只抽了 8 条，其中 1 条含被截断的正确数值，按抽样权重推回去，“更新前低于 Oracle”的符号会反过来。这些回答约 38% 含中日韩文字，自动规则读不了。（人工复核待做；辅助阅读不是人工复核）

### 第十轮：更新强度曲线上的生成效用、IDK 开闸更新的生成审计（规则在运行前写定；没有训练遗忘模型）

37. **按预登记的主口径，很弱的无关更新就让 NPO 在 5 个点的匹配生成预算内披露得比 Oracle 多；但多出来的正确值大多夹在被判为无效的回答里。** 沿 10 个强度不同的更新（lr 1e-4 或 5e-5 × 1、2、5、10、20 步），只用 10 个验证人物为 5 个点预算选出 lr 5e-5 × 5 步。在 40 个测试人物上，NPO 披露正确值的比例高出同更新的 Oracle +0.18\*（27 人为正，Holm 校正后显著）；NPO、Full、Oracle 的 retain 自由生成降幅是 −0.4、3.4、1.6 个点。2 个点预算选出 lr 1e-4 × 2 步：+0.21\*，降幅 0.0、1.7、1.2 个点。保守版本（降幅的 95% 上界也在预算内）都不成立。事后看，这两个更新下 NPO 说出的正确值分别有 70% 和 63% 落在被判为无效的回答里：一句正确的回答后面或中间夹着代码片段、中文句号或其他文字。只算有效回答时，NPO 不高于 Oracle（−0.04 和 −0.01，区间都含 0）。有效回答上的释放要到强度约 4.8 才出现，那里 5 个点的匹配预算已不成立。这类回答算不算披露，要等人工复核。（事先登记，选点只用验证人物；有效回答的分解是事后描述；自动标签）
38. **IDK 在 5 个点的匹配生成预算内没有检测到释放；它的拒答在一段很窄的强度区间里崩塌。** 5 个点预算选出 lr 5e-5 × 5 步：IDK 85% 的回答仍是拒答，披露率与 Oracle 没有可检出的差别（−0.03）。2 个点预算的选点显著低于 Oracle（−0.14\*，100% 拒答）。强度从 2.45 增加到 4.77（lr 5e-5 × 10 步），拒答从 85% 降到 14%，披露率高出 Oracle +0.48\*，40 人全部为正，几乎都是有效回答。这个更新让 IDK 自己的 retain 生成降 4.4 个点，却让 Full 降 7.1、Oracle 降 5.9 个点，所以只在两种情况下可行：10 个点的匹配预算（点估计），以及只看 IDK 自身的 5 个点预算。网格在 2.45 和 4.77 之间没有点，不能写成“5 个点内不可恢复”。（事先登记；只有一个 IDK 种子）
39. **forget-5 上第三轮和第六轮为 IDK 选出的两个开闸更新，在自由生成里都释放，但都不满足 5 个点的生成预算。** 相对三个 Oracle 种子的平均：lr 1e-4 × 5 步 +0.44\*（15 人全部为正），lr 5e-5 × 10 步 +0.35\*（14 人为正）；以单个种子为锚，估计在 +0.34 到 +0.48 之间。IDK 自己的 retain 自由生成分别降 11.6 和 9.4 个点。前者连 10 个点也不可行；后者只在 10 个点、按点估计可行，每种锚点规则都如此，保守版本不可行。第六轮“lr 5e-5 × 10 步在 5 个点预算内（平均锚点）”只对十选一识别成立。第六轮的数字从逐题文件逐位复现。（事先登记；自动标签，这些人物在第三到第八轮用过）
40. **释放对代价的曲线：超出比代价先起来，代价对三个模型相近。** 两个学习率在相同的累积学习率下给出几乎相同的结果，所以曲线的自变量是累积学习率。NPO 的披露率超出在强度 2 左右已有约 +0.2，到 4.9 达到 +0.33，之后持平；retain 自由生成的降幅随强度单调上升，强度 18 时 30–39 个点。Full 和 Oracle 的降幅与遗忘模型相近，强度低时往往更大。网格的 10 个更新里，NPO 和 IDK 各有 5 个满足 5 个点的匹配预算。第九轮的“5 个点生成预算不成立”只对那一个更新成立。（事先登记，曲线是描述；单个 Oracle 种子，第十一轮补了第二个，见第 43 条）

### 第十一轮：forget_10 上的第二个 Oracle 种子和第二个 NPO 训练种子（规则在训练前写定）

41. **NPO 在 forget_10 上的中后层残留和身份绑定，在两个 Oracle 种子和两个 NPO 训练种子下都成立。** 按原配方只换种子，训练了 Oracle 和 NPO 各一个新模型。第 20–28 层只看属性值的读数（fp32 评分），以两个 Oracle 种子的平均为锚：NPO 种子 42 高出 +0.65\*，种子 43 高出 +0.64\*；身份绑定 +0.52\* 和 +0.55\*。以任何一个 Oracle 种子为锚，判定都相同。第二个 NPO 种子同样通过 release gate。（事先登记，测试人物只聚合一次；只有两个种子，没有种子区间）
42. **NPO 更新前的生成行为取决于训练种子，中后层读数不取决于它。**（事先登记；更新前的结果不复现）
    - **更新前：** 种子 42 的披露率比 Oracle 低 0.09\*，回答 99% 是乱码，十选一 0.40；种子 43 高出 +0.07\*，乱码只有 48%，十选一 0.61。
    - **中后层读数：** 两个种子第 20–28 层的读数几乎相同（差 −0.008，区间 −0.10 到 +0.08）。
    - **同一个小更新之后：** 两个种子都高于 Oracle（+0.34\*、+0.21\*），但种子 43 的增量小得多（+0.15\*，种子 42 是 +0.43\*）。在第十轮为 5 个点选出的更新上，种子 43 只高 +0.06\*，与它更新前相同。
    - **对第九轮的影响：** 第九轮的“更新前低于 Oracle”和“99% 乱码”只是种子 42 的特点。
43. **在 forget_10 上，换 Oracle 种子不改变任何判定。** 两个 Oracle 种子在 11 个更新状态上的差：forget 十选一最多 4.6 个点，forget 自由生成 2.3 个点，retain 自由生成 4.4 个点；所有检验的估计都远大于这个差。下面这些在两个 Oracle 种子和它们的平均下都相同：第八到第十轮在 forget_10 上的全部检验；IDK 在第十轮选点上的判定；每一条可行性判定。forget-5 上第三个种子曾改变卡在边界上的结论，所以不能推广成“forget_10 对 Oracle 种子不敏感”。（事先登记；两个 Oracle 种子）

### 对论文的含义（建议）

- **以 Oracle 为锚的评测是方法上的核心**：同一读法、同一更新也施加在 Oracle 上，才能把残留知识和读法或更新本身的效应分开。
- **utility 约束要对三个模型同时成立，而且要先过 release gate。** 只看被攻击模型自己的 utility，会把对健康模型有破坏性的更新当成良性更新；在 release 时就已受损的模型，它的"恢复"要单独标注。
- **utility 要同时看识别和自由生成。** 第九轮里，同一个更新的十选一代价是 3–4 个点，自由生成代价是 7–9 个点，而且丢掉的回答是流畅的错误回答，乱码率看不出来。只按十选一设预算会低估更新的代价；论文里“预算内”要写明是哪一种效用。
- **更新后的生成释放要和它的代价写在同一句里。** 可以写“识别效用保持时目标信息被释放，但 retain 生成有不可忽略的损失”；不能写“保持效用”或“在效用预算下”。论文的数字和措辞以第九轮冻结的主结果表和 claim ledger 为准。第十轮补充：代价随更新强度变化。更弱的更新可以满足 5 个点的生成预算，但那时 NPO 多出来的正确值主要夹在无效回答里；引用“5 个点内的释放”时，必须同时写明更新强度和这一点。
- **输出有效性要用能识别乱码的标签。** 第三轮沿用至今的退化规则只认空白、字符噪声和大量重复，漏掉了混合其他文字和代码片段的输出。引用“通过 release gate”时，要同时给出模型对遗忘人物的无效输出比例。
- **遗忘模型的训练种子要和 Oracle 的种子一样当作变异来源。** 第十一轮里，同一配方只换种子训练的两个 NPO，更新前的输出行为差别很大（乱码 99% 对 48%，十选一 0.40 对 0.61），中后层读数却几乎相同；更新后的释放方向一致，大小相差约 13 个点。forget_10 上 NPO 的结论现在可以写“在两个 Oracle 种子和两个 NPO 训练种子下成立”，不能写种子区间；关于输出行为的说法要写明是哪个种子。
- **拒答式遗忘（IDK）只在完好的知识上加了一层拒答**：知识在 hidden state 里基本完整，闸门在一个对三个模型都很小的良性更新下就会打开。这个闸门在最早几层就被触发、由很多层写入，两组人物上都是如此。应写成 early-triggered, distributed refusal gate，不要写成后层闸门，也不要写触发信号只在图像 token 里。在 Qwen2.5-VL-7B 上，知识同样压在闸门之后，同一个更新也能打开它，但这个更新让 Qwen 的第一个 Oracle 种子降 14 个点，第二个降 5.5 个点。
- **hidden-state 证据以因果干预为准。** 余弦、CKA 这类原始距离分不开知识和其他训练差异；能区分“知识还在”的，是与身份绑定的中层读数，以及“换入 Oracle 的状态后还能不能答对”。逐层 margin 还要用不含模板的读数或按总和打分的十选一复核。第七轮的复核表明，NPO 第 20–28 层的残留不受影响，但输出层的结论要改用只看属性值的读数。第八轮用离线干预直接证明，这种伪影可以在候选证据完全不变时单独产生：主读数应是只看属性值的读数，输出层同时报告按总和打分的十选一，整句平均只用来展示伪影。
- **评分用 fp32 权重和前向，一道题的全部候选同批打分。** fp16 的分块打分没有通过逐 token 的一致性核验，会让约 0.3–0.4% 的十选一判定改变。第七轮原模板下的结论在 fp32 下不变，但论文里对评分的描述要改；第七轮另外两种模板和更早几轮的读数还没有用 fp32 重新打分。
- **审计时把普通无关小更新作为标准攻击。** 按机制定向的攻击在同一预算下都不如它。
- **以三个 Oracle 种子的平均作锚点；卡在显著性或预算边界上的结论同时报告四个锚点。** 这是第六轮事先写定的规则：16 条结论可以直接陈述，29 条要标为对锚点敏感，所有“5 个点预算内”的说法都属于后者。小于约 0.03 的差异落在 Oracle 自身的种子差异之内。Qwen 上也一样：5 个点预算下能否开闸，取决于用哪个 Oracle 作约束。
- **UID 的定义需要再考虑。** 现在的 UID 只比 PUR 少一个与更新无关的常数，按 UID 选和按 PUR 选必然是同一个检查点；要让两者能区分，需要一个随更新变化的基线。
- **IDK 的结论已在第二组人物和第二种架构上检验，Oracle 有三个种子。** Qwen2.5-VL-7B 上只做了 IDK 的最精简审计：知识压在拒答闸门之后、同一个小更新能打开闸门，这两点成立；残留出现的层带不能按相对深度在模型之间换算，应以该模型能读出已知人物知识的深度为参照。NPO 的中后层残留在第八轮也在第二组人物上确认，第十一轮补了第二个 Oracle 种子和第二个 NPO 训练种子，判定不变；GA-Diff 和 KL-Min 的结论仍只来自 LLaVA 的一组人物。Qwen 的 Oracle 有两个种子；遗忘模型里只有 forget_10 的 NPO 有两个训练种子，其余都只有一个。

\* 表示 95% 置信区间不含 0。区间来自按人物聚类的 bootstrap（10,000 次重采样），模型之间的差是配对差。第三轮写"Holm 校正后"的地方，是对 5 个方法 × 3 个预算共 15 个比较做的校正。第三轮的最终测试只有 15 个人物，第四轮和第五轮的 s0 部分沿用这 15 个人物；第五轮的 forget_10 有 40 个测试人物（377 题）。第六轮的 Qwen 审计用的也是这 15 个人物。第七轮的 LLaVA 复核以这 15 人为发现组，另外 10 人（5 个验证人物和 5 个第一轮的重学人物）为复现组，Qwen 部分仍是这 15 个测试人物。第八轮的任务 A、B 用 s0 的 25 个人物（236 题），任务 C 用 forget_10 的 40 个测试人物；这一轮的统计量先按人物平均、再对人物平均，评分用 fp32。第九轮用 forget_10 的 40 个测试人物和 25 个 retain 人物（236 题），同样先按人物平均；自由生成按发布的 fp16 模型运行，回答由事先冻结的自动规则标注，两个检验族各 3 项。第十轮在 forget_10 的 10 个验证人物上选更新、在 40 个测试人物上检验，forget-5 部分用 15 个测试人物；第十一轮用同样的 40 个测试人物，检验族与来源轮次相同，每个锚点分别校正。Holm 校正在每一族检验内部做。

## 相关论文

按与本项目的关系分 6 组，共 59 篇。链接指向 arXiv 摘要页，这里不转载论文全文。

### 本项目使用的模型、基准和遗忘方法

- [Visual Instruction Tuning](https://arxiv.org/abs/2304.08485) — Haotian Liu et al.，NeurIPS 2023。LLaVA：用 GPT-4 生成的图文指令数据端到端微调，是本项目所用模型的来源。
- [Improved Baselines with Visual Instruction Tuning](https://arxiv.org/abs/2310.03744) — Haotian Liu et al.，CVPR 2024。LLaVA-1.5，本项目使用的 7B 基座模型。
- [Protecting Privacy in Multimodal Large Language Models with MLLMU-Bench](https://arxiv.org/abs/2410.22108) — Zheyuan Liu et al.，NAACL 2025。本项目使用的基准：500 个虚构人物和 153 个名人的档案，从多模态和纯文本两方面评测遗忘与效用。
- [TOFU: A Task of Fictitious Unlearning for LLMs](https://arxiv.org/abs/2401.06121) — Pratyush Maini et al.，COLM 2024。虚构作者遗忘基准，给出 GA、GD、KL、IDK 等基线，并以从未见过遗忘集的模型为金标准。本项目的 KL-Min 和 IDK 来自这里。
- [Knowledge Unlearning for Mitigating Privacy Risks in Language Models](https://arxiv.org/abs/2210.01504) — Joel Jang et al.，ACL 2023。对目标序列做梯度上升（GA）来遗忘，是 GA 类方法的来源。
- [Continual Learning and Private Unlearning](https://arxiv.org/abs/2203.12817) — Bo Liu, Qiang Liu, Peter Stone，CoLLAs 2022。TOFU 把梯度差（GD，本项目的 GA-Diff）基线归于这篇论文。
- [Negative Preference Optimization: From Catastrophic Collapse to Effective Unlearning](https://arxiv.org/abs/2404.05868) — Ruiqi Zhang et al.，COLM 2024。NPO：比 GA 更慢走向灾难性崩溃，在遗忘与效用之间平衡得更好。
- [Qwen2.5-VL Technical Report](https://arxiv.org/abs/2502.13923) — Shuai Bai et al.，arXiv 2025。第六轮复现所用的第二种架构 Qwen2.5-VL-7B-Instruct。

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
- [Eliciting Latent Predictions from Transformers with the Tuned Lens](https://arxiv.org/abs/2303.08112) — Nora Belrose et al.，arXiv 2023。为每一层训练一个仿射映射再解码，比 logit lens 更可靠地逐层读出预测。第六轮看到 logit lens 在 Qwen 的中间层连 Oracle 学过的人物都读不出，tuned lens 是可选的替代。
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

`index.html` 是汇总网页，`assets/` 是网页用到的图，`annotate/index.html` 是第九轮人工复核的标注工具。仓库不含实验代码和模型；逐题结果只有标注工具里那 313 条盲化的问答（问题、正确值、模型回答），不含模型、状态和自动标签。论文只给出链接，没有转载 PDF。

## 标注工具（第九轮人工复核）

`annotate/index.html` 是一个单文件网页，给两位标注者使用。

- **内容。** 313 条盲化的模型回答，每条有属性类型、问题、正确值和回答。页面不含模型、状态、抽样层和自动标签；这些在服务器上的 key 文件里，标注完成前不要打开。题目和正确值来自 MLLMU-Bench 的虚构人物档案。
- **打开方式。** 仓库开启 GitHub Pages 后可直接访问 https://gitdrowner.github.io/MMU/annotate/ 。没有开启时，在仓库页面打开 `annotate/index.html`，点 “Download raw file” 下载，再用浏览器打开这个文件即可；页面不联网，也不依赖其他文件。
- **标注。** 进入页面后选“标注者 A”或“标注者 B”（两人看到的是同一批回答，顺序不同），对每条判断 5 个标签：披露正确值、给出矛盾值、拒答、无效输出、流畅但错误。规则在页面右侧，是服务器上 `human_review/annotation_rules_v2.md` 的内容。快捷键：`1`–`5` 切换标签，`Enter` 保存并到下一条，`←` `→` 翻页，`F` 标为待定，`N` 写备注。
- **进度。** 自动保存在本机的浏览器里，不会上传。换电脑、换浏览器或清除浏览器数据之前先点“导出 CSV”，之后用“载入 CSV”继续。
- **提交。** 全部完成后导出 CSV（文件名 `sheet_annotator_A_filled.csv` 或 `…_B_filled.csv`），交给负责人。文件格式就是服务器上评分脚本读取的格式。
- **裁决。** 两份都完成后，在“裁决模式”里载入两份 CSV，页面列出标签不一致的条目，逐条定下最终标签，导出 `sheet_adjudicated.csv`。
- **计算结果。** 把三份 CSV 放到服务器上，运行：

  ```
  python src/r9_review_score.py --tag human --readers <A 的 CSV> <B 的 CSV> --adjudicated <裁决表>
  python src/r9_review_strict.py --tag human --reader <裁决表>
  ```

  第一条给出两人的一致性、自动标签的误差和各模型状态披露率的重估；第二条给出这些标签对结论所用量的含义。
- **请注意。** 两位标注者各自独立完成，标注期间不要讨论具体条目；提交之前请不要阅读汇总页里第九轮的结果。

