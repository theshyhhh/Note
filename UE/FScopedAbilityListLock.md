# FScopedAbilityListLock

## 是什么？

它是 **GAS** 里用于“**在遍历/回调期间临时锁定 Ability 列表**”的一个 **RAII 作用域锁**。作用是：**在锁作用域内避免对 `UAbilitySystemComponent` 的能力列表做结构性修改**（添加/移除），把这些修改**延后**到锁结束后再统一处理，从而防止迭代器失效、重入崩溃与复制状态错乱。官方 API 对它的摘要就是“用于在遍历能力列表时阻止我们移除能力”。

## FScopedAbilityListLock 的工作原理

`FScopedAbilityListLock` 本质上是一个 **RAII (Resource Acquisition Is Initialization)** 类。这意味着：

- 在其**构造函数**中，它会获取对目标Ability列表的锁（内部可能通过临界区等方式实现）。
- 在其**析构函数**中，它会自动释放这个锁。

这种设计确保了即使在锁定期间发生异常，锁也能被正确释放，从而避免了死锁。

## 为什么需要它（典型场景）

1. **遍历激活/可激活能力时**，同时又可能在回调里授予/移除能力（或因为标签/GE改变触发移除）——无锁会导致迭代器失效。
2. **复制/预测回滚**的回调里，可能重新配置能力集——锁能把结构修改推迟到安全点。
3. 你自己写的工具函数在读 `GetActivatableAbilities()` 的同时也会清理某些 Spec——要加锁。

## 用法

```c++
void UAuraAbilitySystemComponent::ForEachAbility(const FForEachAbility& Delegate)
{
	//锁住Ability列表，防止在遍历期间修改列表
	FScopedAbilityListLock AbilityListLock(*this);
	for (const FGameplayAbilitySpec& AbilitySpec : GetActivatableAbilities())
	{
		if (!Delegate.ExecuteIfBound(AbilitySpec))
		{
			UE_LOG(LogAura, Error, TEXT("%hs中委托执行失败"), __FUNCTION__);
		}
	}
}
```

