# 第七章 DQN (进阶技巧)

## 关键词

- **双深度Q网络（double DQN）**：在双深度Q网络中存在两个Q网络，第一个Q网络决定哪一个动作的Q值最大，从而决定对应的动作。另一方面，Q值是用 $Q'$ 计算得到的，这样就可以避免过度估计的问题。具体地，假设我们有两个Q函数并且第一个Q函数高估了它现在执行的动作 $a$ 的值，这没关系，只要第二个Q函数 $Q'$ 没有高估动作 $a$ 的值，那么计算得到的就还是正常的值。

- **竞争深度Q网络（dueling DQN）**：将原来的深度Q网络的计算过程分为两步。第一步计算一个与输入有关的标量 $\mathrm{V(s)}$；第二步计算一个向量 $\mathrm{A(s,a)}$ 对应每一个动作。最后的网络将两步的结果相加，得到我们最终需要的Q值。用一个公式表示就是 $\mathrm{Q(s,a)=V(s)+A(s,a)}$ 。另外，竞争深度Q网络，使用状态价值函数与动作价值函数来评估Q值。

- **优先级经验回放（prioritized experience replay，PER）**：这个方法是为了解决我们在第6章中提出的经验回放方法的不足而提出的。我们在使用经验回放时，均匀地取出回放缓冲区（reply buffer）中的采样数据，这里并没有考虑数据间的权重大小。但是我们应该将那些训练效果不好的数据对应的权重加大，即其应该有更大的概率被采样到。综上，优先级经验回放不仅改变了被采样数据的分布，还改变了训练过程。

- **噪声网络（noisy net）**：其在每一个回合开始的时候，即智能体要和环境交互的时候，在原来的Q函数的每一个参数上加上一个高斯噪声（Gaussian noise），把原来的Q函数变成 $\tilde{Q}$ ，即噪声Q函数。同样，我们把每一个网络的权重等参数都加上一个高斯噪声，就得到一个新的网络 $\tilde{Q}$ 。我们会使用这个新的网络与环境交互直到结束。

- **分布式Q函数（distributional Q-function）**：对深度Q网络进行模型分布，将最终网络的输出的每一类别的动作再进行分布操作。

- **彩虹（rainbow）**：将7个技巧/算法综合起来的方法，7个技巧分别是——深度Q网络、双深度Q网络、优先级经验回放的双深度Q网络、竞争深度Q网络、异步优势演员-评论员算法（A3C）、分布式Q函数、噪声网络，进而考察每一个技巧的贡献度或者与环境的交互是否是正反馈的。


## 习题

**7-1** 为什么传统的深度Q网络的效果并不好？可以参考其公式 $Q(s_t ,a_t)=r_t+\max_{a}Q(s_{t+1},a)$ 来描述。

因为实际应用时，需要让 $Q(s_t ,a_t)$ 与 $r_t+\max_{a}Q(s_{t+1},a)$ 尽可能相等，即与我们的目标越接近越好。可以发现，目标值很容易一不小心就被设置得太高，因为在计算该目标值的时候，我们实际上在做的事情是看哪一个动作 $a$ 可以得到最大的Q值，就把它加上去，使其成为我们的目标。
  
例如，现在有4个动作，本来它们得到的Q值都是差不多的，它们得到的奖励也都是差不多的，但是在估算的时候是有误差的。如果第1个动作被高估了，那目标就会执行该动作，然后就会选这个高估的动作的Q值加上 $r_t$ 当作目标值。如果第4个动作被高估了，那目标就会选第4个动作的Q值加上 $r_t$ 当作目标值。所以目标总是会选那个Q值被高估的动作，我们也总是会选那个奖励被高估的动作的Q值当作Q值的最大值的结果去加上 $r_t$ 当作新目标值，因此目标值总是太大。

**7-2** 在传统的深度Q网络中，我们应该怎么解决目标值太大的问题呢？

我们可以使用双深度Q网络解决这个问题。首先，在双深度Q网络里面，选动作的Q函数与计算价值的Q函数不同。在深度Q网络中，需要穷举所有的动作 $a$，把每一个动作 $a$ 都代入Q函数并计算哪一个动作 $a$ 反馈的Q值最大，就把这个Q值加上 $r_t$ 。但是对于双深度Q网络的两个Q网络，第一个Q网络决定哪一个动作的Q值最大，以此来决定选取的动作。我们的Q值是用 $Q'$ 算出来的，这样有什么好处呢？为什么这样就可以避免过度估计的问题呢？假设我们有两个Q函数，如果第一个Q函数高估了它现在选出来的动作 $a$ 的值，那没关系，只要第二个Q函数 $Q'$ 没有高估这个动作 $a$ 的值，计算得到的就还是正常值。假设反过来是 $Q'$ 高估了某一个动作的值，那也不会产生过度估计的问题。

**7-3** 请问双深度Q网络中所谓的 $Q$ 与 $Q'$ 两个网络的功能是什么？

在双深度Q网络中存在两个Q网络，一个是目标的Q网络，一个是真正需要更新的Q网络。具体实现方法是使用需要更新的Q网络选动作，然后使用目标的Q网络计算价值。双深度Q网络相较于深度Q网络的更改是最少的，它几乎没有增加任何的运算量，甚至连新的网络都不需要。唯一要改变的就是在找最佳动作 $a$ 的时候，本来使用 $Q'$ 来计算，即用目标的Q网络来计算，现在改成用需要更新的Q网络来计算。

**7-4** 如何理解竞争深度Q网络的模型变化带来的好处？

对于 $\mathrm{Q}(s,a)$ ，其对应的状态由于为表格的形式，因此是离散的，而实际中的状态却不是离散的。对于 $\mathrm{Q}(s,a)$ 的计算公式—— $\mathrm{Q}(s,a)=\mathrm{V}(s)+\mathrm{A}(s,a)$ 。其中的 $\mathrm{V}(s)$ 对于不同的状态都有值， $\mathrm{A}(s,a)$ 对于不同的状态都有不同的动作对应的值。所以从本质上来说，我们最终矩阵 $\mathrm{Q}(s,a)$ 的结果是将每一个 $\mathrm{V}(s)$ 加到矩阵 $\mathrm{A}(s,a)$ 中得到的。从模型的角度考虑，我们的网络直接改变的不是 $\mathrm{Q}(s,a)$ ，而是改变的 $\mathrm{V}$、$\mathrm{A}$ 。但是有时我们更新时不一定会将 $\mathrm{V}(s)$ 和 $\mathrm{Q}(s,a)$ 都更新。将状态和动作对分成两个部分后，我们就不需要将所有的状态-动作对都采样一遍，我们可以使用更高效的估计Q值的方法将最终的 $\mathrm{Q}(s,a)$ 计算出来。

**7-5** 使用蒙特卡洛和时序差分平衡方法的优劣分别有哪些？

优势：时序差分方法只采样了一步，所以某一步得到的数据是真实值，接下来的都是Q值估测出来的。使用蒙特卡洛和时序差分平衡方法采样比较多步，如采样$N$步才估测价值，所以估测的部分所造成的影响就会比较小。

劣势：因为智能体的奖励比较多，所以当我们把$N$步的奖励加起来时，对应的方差就会比较大。为了缓解方差大的问题，我们可以通过调整$N$值，在方差与不精确的Q值之间取得一个平衡。这里介绍的参数$N$是超参数，需要微调参数 $N$，例如是要多采样3步、还是多采样5步。

  
## 面试题

**7-1** 友善的面试官：深度Q网络都有哪些变种？引入状态奖励的是哪种？

深度Q网络有3个经典的变种：双深度Q网络、竞争深度Q网络、优先级双深度Q网络。
  
（1）双深度Q网络：将动作选择和价值估计分开，避免Q值被过高估计。

（2）竞争深度Q网络：将Q值分解为状态价值和优势函数，得到更多有用信息。

（3）优先级双深度Q网络：将经验池中的经验按照优先级进行采样。

**7-2** 友善的面试官：请简述双深度Q网络原理。

深度Q网络由于总是选择当前最优的动作价值函数来更新当前的动作价值函数，因此存在过估计问题（估计的价值函数值大于真实的价值函数值）。为了解耦这两个过程，双深度Q网络使用两个价值网络，一个网络用来执行动作选择，然后用另一个网络的价值函数对应的动作值更新当前网络。

**7-3** 友善的面试官：请问竞争深度Q网络模型有什么优势呢？

对于 $\boldsymbol{Q}(s,a)$ ，其对应的状态由于为表格的形式，因此是离散的，而实际的状态大多不是离散的。对于Q值 $\boldsymbol{Q}(s,a)=V(s)+\boldsymbol{A}(s,a)$ 。其中的 $V(s)$ 是对于不同的状态都有值， $\boldsymbol{A}(s,a)$ 对于不同的状态都有不同的动作对应的值。所以本质上，我们最终的矩阵 $\boldsymbol{Q}(s,a)$ 是将每一个 $V(s)$ 加到矩阵 $\boldsymbol{A}(s,a)$ 中得到的。但是有时我们更新时不一定会将 $V(s)$ 和 $\boldsymbol{Q}(s,a)$ 都更新。我们将其分成两个部分后，就不需要将所有的状态-动作对都采样一遍，我们可以使用更高效的估计Q值的方法将最终的 $\boldsymbol{Q}(s,a)$ 计算出来。
