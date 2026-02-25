
# UE5 常用调试 API 

## 基础调试输出

### 日志输出
```cpp
// 基础日志
UE_LOG(LogTemp, Log, TEXT("This is a log message"));
UE_LOG(LogTemp, Warning, TEXT("This is a warning"));
UE_LOG(LogTemp, Error, TEXT("This is an error"));

// 带变量的输出
float Health = 75.0f;
UE_LOG(LogTemp, Log, TEXT("Player health: %f"), Health);

// 带分类的日志（需先在头文件声明）
DECLARE_LOG_CATEGORY_EXTERN(LogMyGame, Log, All);
DEFINE_LOG_CATEGORY(LogMyGame);
UE_LOG(LogMyGame, Log, TEXT("Custom category log"));
// 屏幕输出（默认显示2秒）
GEngine->AddOnScreenDebugMessage(-1, 2.0f, FColor::White, TEXT("Hello World!"));

// 带Key的屏幕输出（可更新）
static int32 MyDebugKey = 0;
GEngine->AddOnScreenDebugMessage(MyDebugKey, 5.0f, FColor::Green, 
    FString::Printf(TEXT("Health: %.2f"), Health));





// 绘制调试球体
DrawDebugSphere(GetWorld(), Location, Radius, Segments, Color, bPersistentLines, LifeTime, DepthPriority, Thickness);

// 绘制调试盒子
DrawDebugBox(GetWorld(), Center, Extent, Color, bPersistentLines, LifeTime, DepthPriority, Thickness);

// 绘制调试线条
DrawDebugLine(GetWorld(), Start, End, Color, bPersistentLines, LifeTime, DepthPriority, Thickness);

// 绘制调试点
DrawDebugPoint(GetWorld(), Location, Size, Color, bPersistentLines, LifeTime, DepthPriority)；
 ```




##  Unreal Engine 5 常用宏速查表

### **1. 类声明宏**

| 宏                   | 用途                 | 示例                                                     |
| ------------------- | ------------------ | ------------------------------------------------------ |
| `UCLASS()`          | 声明一个 UE 反射类        | `UCLASS(Blueprintable, meta=(DisplayName="My Actor"))` |
| `UINTERFACE()`      | 声明一个 UE 反射接口       | `UINTERFACE(MinimalAPI, Blueprintable)`                |
| `UGENERATED_BODY()` | 自动生成类反射代码（必须放在类体内） | `GENERATED_BODY()`                                     |

---

### **2. 属性宏 (`UPROPERTY`)**

|宏|用途|示例|
|---|---|---|
|`BlueprintReadOnly`|蓝图只读|`UPROPERTY(BlueprintReadOnly)`|
|`BlueprintReadWrite`|蓝图可读写|`UPROPERTY(BlueprintReadWrite)`|
|`EditAnywhere`|在编辑器任意位置可编辑|`UPROPERTY(EditAnywhere)`|
|`EditDefaultsOnly`|仅可在默认值（CDO）编辑|`UPROPERTY(EditDefaultsOnly)`|
|`VisibleAnywhere`|在编辑器可见但不可编辑|`UPROPERTY(VisibleAnywhere)`|
|`Category="CategoryName"`|在编辑器分类|`UPROPERTY(Category="Movement")`|
|`meta=(DisplayName="Nice Name")`|显示更友好的名称|`UPROPERTY(meta=(DisplayName="Health"))`|
|`Replicated`|网络同步（需在 `GetLifetimeReplicatedProps` 处理）|`UPROPERTY(Replicated)`|
|`Transient`|不保存到磁盘（临时变量）|`UPROPERTY(Transient)`|
|`SaveGame`|可序列化到存档|`UPROPERTY(SaveGame)`|

---

### **3. 函数宏 (`UFUNCTION`)**

|宏|用途|示例|
|---|---|---|
|`BlueprintCallable`|可在蓝图中调用|`UFUNCTION(BlueprintCallable)`|
|`BlueprintPure`|纯函数（无副作用）|`UFUNCTION(BlueprintPure)`|
|`BlueprintImplementableEvent`|蓝图可覆盖的虚函数（无 C++ 实现）|`UFUNCTION(BlueprintImplementableEvent)`|
|`BlueprintNativeEvent`|蓝图可覆盖的虚函数（有默认 C++ 实现）|`UFUNCTION(BlueprintNativeEvent)`|
|`Server`|仅在服务器执行（RPC）|`UFUNCTION(Server, Reliable)`|
|`Client`|仅在客户端执行（RPC）|`UFUNCTION(Client, Reliable)`|
|`NetMulticast`|多播（所有客户端执行）|`UFUNCTION(NetMulticast, Reliable)`|
|`WithValidation`|RPC 参数验证|`UFUNCTION(Server, WithValidation)`|

---

### **4. 结构体宏 (`USTRUCT`)**

|宏|用途|示例|
|---|---|---|
|`USTRUCT()`|声明反射结构体|`USTRUCT(BlueprintType)`|
|`GENERATED_BODY()`|生成结构体反射代码|`GENERATED_BODY()`|
|`BlueprintType`|可在蓝图中使用|`USTRUCT(BlueprintType)`|
|`meta=(BlueprintInternalUseOnly)`|仅限内部使用|`USTRUCT(meta=(BlueprintInternalUseOnly))`|

---

### **5. 枚举宏 (`UENUM`)**

|宏|用途|示例|
|---|---|---|
|`UENUM()`|声明反射枚举|`UENUM(BlueprintType)`|
|`BlueprintType`|可在蓝图中使用|`UENUM(BlueprintType)`|
|`meta=(DisplayName="Nice Name")`|显示友好名称|`UENUM(meta=(DisplayName="Weapon Type"))`|

---

### **6. 其他常用宏**

| 宏                                    | 用途         | 示例                                                                       |
| ------------------------------------ | ---------- | ------------------------------------------------------------------------ |
| `DECLARE_DYNAMIC_MULTICAST_DELEGATE` | 声明动态多播委托   | `DECLARE_DYNAMIC_MULTICAST_DELEGATE(FMyDelegate);`                       |
| `DECLARE_DELEGATE`                   | 声明单播委托     | `DECLARE_DELEGATE(FMySimpleDelegate);`                                   |
| `UE_LOG`                             | 打印日志       | `UE_LOG(LogTemp, Warning, TEXT("Hello"));`                               |
| `ensure()`                           | 运行时检查（不崩溃） | `ensure(MyPointer != nullptr);`                                          |
| `check()`                            | 运行时断言（崩溃）  | `check(MyPointer != nullptr);`                                           |
| `GEngine->AddOnScreenDebugMessage`   | 屏幕打印调试信息   | `GEngine->AddOnScreenDebugMessage(-1, 5.f, FColor::Red, TEXT("Hello"));` |
|                                      |            |                                                                          |
## UCapsuleComponent 胶囊体 常用 API 速查表

### **1. 基础属性设置**

| **API**                                                                | **说明**              | **示例**                                                          |
| ---------------------------------------------------------------------- | ------------------- | --------------------------------------------------------------- |
| `SetCapsuleSize(float Radius, float HalfHeight, bool bUpdateOverlaps)` | 设置胶囊体的半径和半高         | `CapsuleComp->SetCapsuleSize(50.f, 100.f);`                     |
| `GetScaledCapsuleRadius()`                                             | 获取当前缩放后的半径          | `float Radius = CapsuleComp->GetScaledCapsuleRadius();`         |
| `GetScaledCapsuleHalfHeight()`                                         | 获取当前缩放后的半高          | `float HalfHeight = CapsuleComp->GetScaledCapsuleHalfHeight();` |
| `GetCapsuleAxis()`                                                     | 获取胶囊体的朝向轴（默认 `Z` 轴） | `FVector Axis = CapsuleComp->GetCapsuleAxis();`                 |

---

### **2. 碰撞与物理**

|**API**|**说明**|**示例**|
|---|---|---|
|`SetCollisionEnabled(ECollisionEnabled::Type NewType)`|启用/禁用碰撞|`CapsuleComp->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);`|
|`SetCollisionResponseToChannel(ECollisionChannel Channel, ECollisionResponse Response)`|设置对特定通道的碰撞响应|`CapsuleComp->SetCollisionResponseToChannel(ECC_Pawn, ECR_Ignore);`|
|`SetGenerateOverlapEvents(bool bGenerate)`|是否生成重叠事件|`CapsuleComp->SetGenerateOverlapEvents(true);`|
|`OnComponentBeginOverlap.AddDynamic()`|绑定重叠开始事件|`CapsuleComp->OnComponentBeginOverlap.AddDynamic(this, &AMyActor::OnCapsuleBeginOverlap);`|
|`OnComponentEndOverlap.AddDynamic()`|绑定重叠结束事件|`CapsuleComp->OnComponentEndOverlap.AddDynamic(this, &AMyActor::OnCapsuleEndOverlap);`|

---

### **3. 位置与变换**

|**API**|**说明**|**示例**|
|---|---|---|
|`GetComponentLocation()`|获取胶囊体中心位置|`FVector Location = CapsuleComp->GetComponentLocation();`|
|`GetComponentRotation()`|获取胶囊体旋转|`FRotator Rotation = CapsuleComp->GetComponentRotation();`|
|`SetWorldLocation(FVector NewLocation)`|设置世界坐标位置|`CapsuleComp->SetWorldLocation(FVector(0, 0, 100));`|
|`AddLocalOffset(FVector DeltaLocation)`|局部坐标偏移|`CapsuleComp->AddLocalOffset(FVector(10, 0, 0));`|

---

### **4. 调试与可视化**

|**API**|**说明**|**示例**|
|---|---|---|
|`SetHiddenInGame(bool bHidden)`|隐藏/显示胶囊体|`CapsuleComp->SetHiddenInGame(false);`|
|`ShapeColor`|设置调试颜色（编辑器可见）|`CapsuleComp->ShapeColor = FColor::Green;`|
|`bDrawOnlyIfSelected`|仅在选中时显示|`CapsuleComp->bDrawOnlyIfSelected = true;`|
|`MarkRenderStateDirty()`|强制刷新渲染状态|`CapsuleComp->MarkRenderStateDirty();`|

---

### **5. 射线检测与几何查询**

| **API**                                                                             | **说明**        | **示例**                                                                            |
| ----------------------------------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------- |
| `IsOverlappingComponent(UPrimitiveComponent* OtherComp)`                            | 检查是否与其他组件重叠   | `bool bOverlapping = CapsuleComp->IsOverlappingComponent(OtherActor->GetMesh());` |
| `SweepComponent(FHitResult& OutHit, FVector Start, FVector End, FRotator Rotation)` | 胶囊体扫描检测       | `CapsuleComp->SweepComponent(OutHit, Start, End, FQuat::Identity);`               |
| `GetOverlappingActors(TArray<AActor*>& OutActors)`                                  | 获取所有重叠的 Actor | `CapsuleComp->GetOverlappingActors(OverlappingActors);`                           |
|                                                                                     |               |                                                                                   |
# 零散重要api
**AutoPossessPlayer =EAutoReceiveInput::player;     设置玩家控制器



# UE5.5 增强输入系统使用指南 (C++ 实现)

以下是在 Unreal Engine 5.5 中使用增强输入系统(Enhanced Input System)的完整 C++ 实现方式，内容已格式化便于复制到 Obsidian。

## 1. 基本设置

### 1.1 启用增强输入模块

```
首先在项目的 `Build.cs` 文件中添加依赖：
PublicDependencyModuleNames.AddRange(new string[] { 
    "Core", 
    "CoreUObject", 
    "Engine", 
    "InputCore", 
    "EnhancedInput" 
});
```
### 1.2 创建输入动作(Input Actions)

在内容浏览器中右键创建 `Input Actions` 资源：

- `IA_Jump`
    
- `IA_Move`
    
- `IA_Look`
    
- `IA_Fire`

## 2. C++ 实现

### 2.1 玩家控制器头文件 (MyPlayerController.h)
```
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "InputActionValue.h"
#include "MyPlayerController.generated.h"

class UInputMappingContext;
class UInputAction;

UCLASS()
class MYPROJECT_API AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

protected:
    virtual void BeginPlay() override;
    virtual void SetupInputComponent() override;

    // 输入映射上下文
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    UInputMappingContext* DefaultMappingContext;

    // 输入动作
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    UInputAction* JumpAction;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    UInputAction* MoveAction;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    UInputAction* LookAction;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
    UInputAction* FireAction;

private:
    // 输入回调函数
    void OnJump(const FInputActionValue& Value);
    void OnMove(const FInputActionValue& Value);
    void OnLook(const FInputActionValue& Value);
    void OnFire(const FInputActionValue& Value);
};
```
### 2.2 玩家控制器实现文件 (MyPlayerController.cpp)
```
#include "MyPlayerController.h"
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "GameFramework/Character.h"

void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();

    // 获取本地玩家增强输入子系统并添加映射上下文
    if (UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetLocalPlayer()))
    {
        Subsystem->AddMappingContext(DefaultMappingContext, 0);
    }
}

void AMyPlayerController::SetupInputComponent()
{
    Super::SetupInputComponent();

    // 设置增强输入组件
    if (UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(InputComponent))
    {
        EnhancedInputComponent->BindAction(JumpAction, ETriggerEvent::Triggered, this, &AMyPlayerController::OnJump);
        EnhancedInputComponent->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyPlayerController::OnMove);
        EnhancedInputComponent->BindAction(LookAction, ETriggerEvent::Triggered, this, &AMyPlayerController::OnLook);
        EnhancedInputComponent->BindAction(FireAction, ETriggerEvent::Triggered, this, &AMyPlayerController::OnFire);
    }
}

void AMyPlayerController::OnJump(const FInputActionValue& Value)
{
    if (ACharacter* Character = GetCharacter())
    {
        Character->Jump();
    }
}

void AMyPlayerController::OnMove(const FInputActionValue& Value)
{
    const FVector2D MovementVector = Value.Get<FVector2D>();
    
    if (ACharacter* Character = GetCharacter())
    {
        const FRotator Rotation = GetControlRotation();
        const FRotator YawRotation(0, Rotation.Yaw, 0);
        
        const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
        const FVector RightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
        
        Character->AddMovementInput(ForwardDirection, MovementVector.Y);
        Character->AddMovementInput(RightDirection, MovementVector.X);
    }
}

void AMyPlayerController::OnLook(const FInputActionValue& Value)
{
    const FVector2D LookAxisVector = Value.Get<FVector2D>();
    
    if (IsLookInputIgnored())
    {
        return;
    }
    
    AddYawInput(LookAxisVector.X);
    AddPitchInput(LookAxisVector.Y);
}

void AMyPlayerController::OnFire(const FInputActionValue& Value)
{
    // 实现射击逻辑
    UE_LOG(LogTemp, Warning, TEXT("Fire!"));
}
```
# 如何获取控制器的旋转角度并转换成向量
```
void ABird::Move(float Value)
{
	UE_LOG(LogTemp, Warning, TEXT("MoveForward"));
    if (Value != 0.0f)
    {
       //根据控制器旋转获取移动方向
        FRotator Rotation = GetControlRotation();
        FRotator YawRotation(0, Rotation.Yaw, 0);//获取旋转的Yaw角度
        FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);//通过旋转矩阵获取方向向量
        AddMovementInput(Direction, Value);





    }
}、
```


# 添加模块的办法
```
PublicDependencyModuleNames.AddRange(
    new string[] {
        "Core",
        "CoreUObject",
        "Engine",
        "GroomComponent"  // 添加毛发组件模块
    }
);
```
# 动画蓝图
## 要获取父类的数据要使用强制转换，将子类对象转换为父类对象    
Cast<类名> (TryGetPawnOwner());