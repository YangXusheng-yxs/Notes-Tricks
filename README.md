## 记录pytorch中BN的几个坑

最近参考对比学习中的Siamese network实现提取表征的功能，其中Projection network和Prediction network用到了Batchnorm1d layer，但是eval模式下结果远差于train模式（valid_loss直接形如nike的logo起飞了。。），让我很费解，自己统计了训练集和验证集的均值和方差可以认为是同分布的，因此推理大概率不是数据本身的问题而是代码环节出了问题。这个Layer有几点需要注意的，特此记录几个坑。

首先，必须提一下Pytorch的社区维护还是蛮好的，有很多小坑都会给出专业解答，对用户很有帮助。Pytorch中的model.train()和model.eval()也是十分不错的设计，需要正确结合batchnorm一起使用。主要参考了几篇blog和社区response。

1.具体batchnorm的原理参考这篇[[blog]](https://zhuanlan.zhihu.com/p/232487440)。如果想手动实现加深理解batchnorm那么可以参考[[这篇blog]](https://www.jianshu.com/p/2b94da24af3b)。

2.参考了[[这篇blog]](https://cloud.tencent.com/developer/article/1725409)总结的所有方式,有一点启发但还是没有解决我的问题,几个tips可以尝试:

- 调大了batchsize，这样的目的是为了让统计值不会由于batchsize过小导致计算偏差；
- 降低了BN层的momentum，那么训练集存储的running_mean就会更新的幅度较小，和上一次比较接近；
- 检查是否存在不同的层使用相同BN层的bug；
- 尝试去掉eval模式或者在BN层设置track_running_stats为False，相当于切回train模式罢了，所以也不对。


3.这位pytorch维护者**[[ptrblck]](https://discuss.pytorch.org/u/ptrblck)**在这篇[[这篇blog]](https://discuss.pytorch.org/t/using-dataloader-yields-different-results-for-shuffle-true-false/59375/6)回答中提到了model.train()和model.eval()的使用。

- If you don’t switch to .eval() before using the evaluation dataset (or test set), the batch norm stats will be updated, so your model will “train” using this dataset, which is considered a data leakage.
Shuffling the validation or test dataset might even result in “better” stats to leak.
- Note that shuffling does not change anything, if you set the model properly to eval() before validating the model.

4.同样这位pytorch维护者还给出了Batchnorm2d的使用官方回复。code的地址在[[Github地址]](https://github.com/ptrblck/pytorch_misc/blob/master/batch_norm_manual.py#L62)

5.最后,[[这篇blog]](https://blog.csdn.net/qq_38556984/article/details/105662342)中的一句话仔细读了几次,提醒到了自己.**问题的关键:**BN层的统计数据更新是在每一次训练的阶段的model.train()后的forward方法中自动实现的,而不是在梯度计算与反向传播中更新optim.step()中完成的。
所以说即使不计算梯度，整个模型还是由于BN的变化更新了。

这句话的意思就是说每一次调用forward()，那么BN的统计量就更新了。忽然想到自己实现对比学习用了triplet loss，那么我需要分别得到anchor，positive和negative的embedding，我是分别过的network，这就是问题的关键了。。恍然大悟！将数据一次性forword()，正常加上model.train()和model.eval()结果就正常了。之前使用Batchnorm Layer都是一次性forward()，所以没有在意这个问题。也理解为什么很多对比学习代码实现把数据集一起过network。

**切记：当使用batchnorm推荐使用model.train()和model.eval()来区分**。batchnorm中train模式是每个batch训练集的统计量，保证BN用每一批数据的均值和方差，然后利滑动平均得到mean, var。α，β参数是通过网络学习出来的，可以理解为一个Linear层。在test模式下，由于此时的batchsize大小可能很小，因此不在使用当前的batch的均值和方差，仅使用全部训练数据的均值和方差。

不过，就是加了batchnorm之后性能没有特别的提升，收敛速度确实有一些提高。但至少掌握了batchnorm的正确打开方式。














model.eval()来区分







































  