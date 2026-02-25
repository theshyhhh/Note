#  UBlueprintAsyncActionBase

## 是什么？

这是所有蓝图层异步动作的基类；**继承它 + BlueprintCallable 工厂函数 → 自动生成特殊异步节点**

## 特点

非常轻量，就是一个 UObject + 委托

不需要任务管理器、没有优先级/资源系统

蓝图调用方式直观：“一个节点 + 一堆 delegate 输出”

## 生命周期 & 常见坑

- UE 会持有这个 AsyncAction 节点的引用，直到：
  - 节点完成（你广播完事件之后），并且
  - 内部逻辑调用 `SetReadyToDestroy` 或没有任何其他引用时被 GC
- 很多教程建议在完成时 *手动释放*：比如包装一个 `EndAction()` 调用，里面取消 timer、解绑回调、清理引用后调用 `SetReadyToDestroy()`。
- 千万不要忘了：
  - 避免循环引用（例如把 Owner 存为 UPROPERTY 的同时，Owner 又存你）
  - 不要在已经被 GC 标记之后再广播委托

## 简单使用

用于向UI传递对应Tag技能进出CD以及CD时长

```c++
#pragma once

#include "CoreMinimal.h"
#include "ActiveGameplayEffectHandle.h"
#include "GameplayTagContainer.h"
#include "Kismet/BlueprintAsyncActionBase.h"
#include "WaitCooldownChange.generated.h"

struct FGameplayEffectSpec;
class UAbilitySystemComponent;

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnCooldownChangedSiganature, float, Time);

UCLASS()
class AURA_API UWaitCooldownChange : public UBlueprintAsyncActionBase
{
	GENERATED_BODY()

public:
	UPROPERTY(BlueprintAssignable)
	FOnCooldownChangedSiganature CooldownStart;

	UPROPERTY(BlueprintAssignable)
	FOnCooldownChangedSiganature CooldownEnd;

	//工厂函数
	UFUNCTION(BlueprintCallable, meta=(BlueprintInternalUseOnly="true"))
	static UWaitCooldownChange* WaitCooldownChange(UAbilitySystemComponent* AbilitySystemComponent, const FGameplayTag& CooldownTag);

	UFUNCTION(BlueprintCallable)
	void EndTask();

protected:
	UPROPERTY()
	TObjectPtr<UAbilitySystemComponent> ASC;

	FGameplayTag CDTag;

	void CooldownTagChanged(const FGameplayTag Tag, int32 NewCount);
	void OnActiveEffectAdded(UAbilitySystemComponent* AbilitySystemComponent, const FGameplayEffectSpec& EffectSpec,
	                         FActiveGameplayEffectHandle EffectHandle);
};

```

```C++
#include "AbilitySystem/AysncTask/WaitCooldownChange.h"

#include "AbilitySystemComponent.h"

UWaitCooldownChange* UWaitCooldownChange::WaitCooldownChange(UAbilitySystemComponent* AbilitySystemComponent, const FGameplayTag& CooldownTag)
{
	UWaitCooldownChange* WaitCooldownChange = NewObject<UWaitCooldownChange>();
	if (!IsValid(AbilitySystemComponent) || !CooldownTag.IsValid())
	{
		WaitCooldownChange->EndTask();
		return nullptr;
	}
	WaitCooldownChange->ASC = AbilitySystemComponent;
	WaitCooldownChange->CDTag = CooldownTag;
	WaitCooldownChange->ASC->RegisterGameplayTagEvent(WaitCooldownChange->CDTag, EGameplayTagEventType::NewOrRemoved).AddUObject(
		WaitCooldownChange, &UWaitCooldownChange::CooldownTagChanged);
	WaitCooldownChange->ASC->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(WaitCooldownChange, &UWaitCooldownChange::OnActiveEffectAdded);
	return WaitCooldownChange;
}

void UWaitCooldownChange::EndTask()
{
	if (!IsValid(ASC))return;
	ASC->RegisterGameplayTagEvent(CDTag, EGameplayTagEventType::NewOrRemoved).RemoveAll(this);

	SetReadyToDestroy();
	MarkAsGarbage();
}

void UWaitCooldownChange::CooldownTagChanged(const FGameplayTag Tag, int32 NewCount)
{
	if (NewCount == 0)
	{
		CooldownEnd.Broadcast(0.f);
	}
}

void UWaitCooldownChange::OnActiveEffectAdded(UAbilitySystemComponent* AbilitySystemComponent, const FGameplayEffectSpec& EffectSpec,
                                              FActiveGameplayEffectHandle EffectHandle)
{
	FGameplayTagContainer AssetTags;
	EffectSpec.GetAllAssetTags(AssetTags);
	FGameplayTagContainer GrantedTags;
	EffectSpec.GetAllGrantedTags(GrantedTags);
	if (AssetTags.HasTagExact(CDTag) || GrantedTags.HasTagExact(CDTag))
	{
		FGameplayEffectQuery GEQuery = FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags(CDTag.GetSingleTagContainer());
		TArray<float> Time = ASC->GetActiveEffectsTimeRemaining(GEQuery);
		if (Time.Num() > 0)
		{
			CooldownStart.Broadcast(Time[0]);
		}
	}
}

```

![image-20251114113329300](D:\学习\学习笔记\UE\Image\image-20251114113329300.png)