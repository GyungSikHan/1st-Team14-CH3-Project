# ⚡ 30초 요약 (TL;DR)

# 🎥 게임 플레이 영상
<p align="center">
  <a href="https://youtu.be/tJcozATuAgQ">
    <img src="image.png" width="1000">
  </a>
</p>

# 🎮 포트폴리오 링크
https://drive.google.com/file/d/1wuGg5KNYLVXD0ZXs_h4th62TAfnBNP6L/view?usp=sharing

# 📌 프로젝트 소개

"**SymBio**"는 언리얼 엔진 기반으로 제작된 1인칭 FPS 게임으로, 극한의 생존을 위해 인간과 SymBio가 정신적·신체적으로 융합된 세계관을 담고 있습니다. 플레이어는 제한된 자원과 능력 속에서 전투와 생존을 동시에 수행해야 합니다.

# 🎮 게임 개발
> 
> - **인원**: 4인
> - **기간**: 25.02.17 ~ 25.03.07
> - **목적**: 언리얼 엔진의 **Gameplay Framework**와 **Component 기반 구조**를 이해하고, C++과 Blueprint 기반으로 FPS 게임의 핵심 시스템(무기, 캐릭터, 데미지)을 직접 구현하는 경험을 목표로 함.
> - **기술**: C++, Unreal Engine 5.5, Blueprint, Git, Slack, Rider/Visual Studio
</aside>

## 🖼 In-Game Screenshot

![alt text](image-1.png)
![alt text](image-2.png)
---

# **👨‍💻 My Key Contributions**

## Component
### ✔ 설계 의도
- 기능을 재사용하기 쉽고, 분리, 조합하여 Actor를 가볍고 유연하게 만들기 위해 사용

### ✔ 구현 내용
#### ↳ [WeaponComponent](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Components/CWeaponComponent.cpp)

- [무기 타입 판단](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Components/CWeaponComponent.cpp#L58-L160)
	- 무기 장착 및 해제 트리거 발동시 현재 장착된 무기의 타입과 들어온 타입의 무기를 비교하여 같으면 장착 해제, 다르면 새로들어온 타입으로 무기를 교체하도록 구현

```cpp
void UCWeaponComponent::SetUnarmedMode()
{
	if (GetCurrentWeapon()->CanUnequip() == false)
		return;
	GetCurrentWeapon()->Unequip();
	ChangeType(EWeaponType::Max);

	// 무기가 해제되었으므로 HUD에 무기 이름을 비움
	OnWeaponNameChanged.Broadcast(FText::GetEmpty());

}

void UCWeaponComponent::SetRifleMode()
{
	int32 count{};
	for(auto a : Weapons)
	{
		if (a->GetItemType() == EItemType::EIT_Rifle)
			count++;
	}
	if(count == 0)
		return;
	SetMode(EWeaponType::Rifle);
}

void UCWeaponComponent::SetKnifeMode()
{
	int32 count{};
	for (auto a : Weapons)
	{
		if (a->GetItemType() == EItemType::EIT_Knife)
			count++;
	}
	if (count == 0)
		return;

	SetMode(EWeaponType::Knife);
}

void UCWeaponComponent::SetGrenadeMode()
{
	SetMode(EWeaponType::Grenade);
}

void UCWeaponComponent::SetPistolMode()
{
	int32 count{};
	for (auto a : Weapons)
	{
		if (a->GetItemType() == EItemType::EIT_Pistol)
			count++;
	}
	if (count == 0)
		return;
	SetMode(EWeaponType::Pistol);
}

void UCWeaponComponent::SetMode(EWeaponType InType)
{
	if (Type == InType)
	{
		SetUnarmedMode();
		return;
	}
	else if(IsUnarmedMode() == false)
	{
		//무기를 장착하고 있는 상태라면 현재 무기를 장착해제할 수 있는지 체크한뒤 무기 장착 해제
		if(GetCurrentWeapon()->CanUnequip() == false)
			return;
		GetCurrentWeapon()->Unequip();
	}

	if (Weapons[(int32)InType] == nullptr)
		return;
	if (Weapons[(int32)InType]->CanEquip() == false)
		return;
	Weapons[(int32)InType]->Equip();
	ChangeType(InType);

	if (ACWeapon* CurrentWeapon = GetCurrentWeapon())
	{
		FText WeaponName = CurrentWeapon->GetWeaponDisplayName();
		OnWeaponNameChanged.Broadcast(WeaponName);
	}
}

```

- [무기에게 명령 내리기](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Components/CWeaponComponent.cpp#L171-L253)
	- 현재 장착 무기에게 명령을 내리는 함수들로 몽타주에 AnimNotify에서 호출하도록 구현
	- 이를 통해 몽타주 등이 실행될 때 총알 발사, 카메라 효과, Two Bone IK등이 실행

```cpp
void UCWeaponComponent::Begin_Equip()
{
	if (GetCurrentWeapon() == nullptr)
		return;
	GetCurrentWeapon()->BeginEquip();
}

void UCWeaponComponent::End_Equip()
{
	if (GetCurrentWeapon() == nullptr)
		return;
	GetCurrentWeapon()->EndEquip();
}

void UCWeaponComponent::DoAction()
{if(GetCurrentWeapon() == nullptr)
		return;
	if (IsRifleMode() == true || IsPistolMode() == true)
	{
		Begin_Fire();
		return;
	}

	GetCurrentWeapon()->DonAction();
}

void UCWeaponComponent::Begin_DoAction()
{
	if(GetCurrentWeapon() == nullptr)
		return;
	GetCurrentWeapon()->BeginAction();
}

void UCWeaponComponent::End_DoAction()
{
	if (GetCurrentWeapon() == nullptr)
		return;
	GetCurrentWeapon()->EndAction();
}

void UCWeaponComponent::Begin_Fire() // 무기 발사 시작
{
	if (GetCurrentWeapon() == nullptr)
		return;
	if (GetCurrentWeapon()->CanFire() == false)
		return;

	GetCurrentWeapon()->BeginFire();
	...
}

void UCWeaponComponent::End_Fire() // 무기 발사 종료
{
	if (GetCurrentWeapon() == nullptr)
		return;

	GetCurrentWeapon()->EndFire();

	...
}

bool UCWeaponComponent::BeginAim()
{
	if (GetCurrentWeapon() == nullptr)
		return false;
	if(GetCurrentWeapon()->CanAim() == false)
		return false;

	GetCurrentWeapon()->BeginAim();
	OnAimChanged.Broadcast(true); // Aim 상태 변경(줌인) 델리게이트 호출
	return true;
}

void UCWeaponComponent::EndAim()
{
	if (GetCurrentWeapon() == nullptr)
		return;

	GetCurrentWeapon()->EndAim();
	OnAimChanged.Broadcast(false); // Aim 상태 변경(줌아웃) 델리게이트 호출
}

void UCWeaponComponent::Eject_Magazine()
{
	if (GetCurrentWeapon() == nullptr)
		return;

	GetCurrentWeapon()->Eject_Magazine();
}

void UCWeaponComponent::Spawn_Magazine()
{
	if (GetCurrentWeapon() == nullptr)
		return;

	GetCurrentWeapon()->Spawn_Magazine();
}

void UCWeaponComponent::Load_Magazine()
{
	if (GetCurrentWeapon() == nullptr)
		return;

	GetCurrentWeapon()->Load_Magazine();
}

void UCWeaponComponent::End_Magazine()
{
	if (GetCurrentWeapon() == nullptr)
		return;

	GetCurrentWeapon()->End_Magazine();
}

void UCWeaponComponent::On_Begin_Aim(ACWeapon* InThisWeapon)
{
	if(GetCurrentWeapon() == nullptr)
		return;
	for (ACWeapon* weapon : Weapons)
	{
		if (weapon != InThisWeapon)
			weapon->SetHidden(true);
		else
			weapon->SetHidden(false);
	}
}

void UCWeaponComponent::On_End_Aim()
{
	if (GetCurrentWeapon() == nullptr)
		return;
	for (ACWeapon* weapon : Weapons)
			weapon->SetHidden(false);
}
```
#### ↳ [StatusComponent](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Components/CStatusComponent.cpp)
- Character들의 데미지, 스테미너, 회복 등을 컴포넌트를 통해 관리하도록 구현

```cpp
void UCStatusComponent::Damage(float Amount)
{
	Health += (Amount * -1);
	Health = FMath::Clamp(Health, 0.0f, MaxHealth);
}

void UCStatusComponent::UseStamina(float Amount)
{
	Stamina +=(Amount *-1);
	Stamina = FMath::Clamp(Stamina, 0.0f, MaxStamina);
}

void UCStatusComponent::HealHealth(float Amount)
{
	Health += Amount;
	Health = FMath::Clamp(Health, 0.0f, MaxHealth);
}

void UCStatusComponent::HealStamina(float Amount)
{
	Stamina += Amount;
	Stamina = FMath::Clamp(Stamina, 0.0f, MaxStamina);
}
```

#### ↳ 그 외에 Component
- [MontagesComponent](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Components/CMontagesComponent.cpp)
	- Character들의 기본 몽타주들을 관리하는 컴포넌트로, 데이터 테이블에서 값을 읽어와 해당 타입의 몽타주가 실행되도록 구현
- [CameraComponent](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Components/CCameraComponent.cpp)
	- 카메라를 전반적으로 관리하는 Component
	- 해당 Component를 이용하여 Camera를 가져오지 않고 제어가 가능하도록 구현
- [MovementComponent](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Components/CMovementComponent.cpp)
	- Player의 움직임을 전반적으로 관리하는 Component
	- 해당 Component를 이용하여 Character들의 움직임을 제어 할 수 있음
- [StateComponent](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Components/CStateComponent.cpp)
	- Character의 상태를 관리하는 Component
	- Character의 상태에 따른 상태이상, 행동이 끝나지 않은 상태에서 다른 행동을 하지 못하도록 막는 등의 역할을 하도록 구현


## 🔫 **Weapon System**

### ✔ 설계 의도

- 무기 종류에 따른 로직 중복을 줄이고 유지 보수성을 높이기 위해 **Weapon Base Class(ACWeapon)**를 설계하여 공통 기능을 통합하여 코드 재사용성 향상
- 장착/조준/발사/재장전/피격 연출 등 무기의 필수 기능을 하나의 베이스에서 일관성 있게 관리하여 무기 확장에 용이
- 각 무기 별 필요 기능은 Override로 구성할 수 있도록 설계하여 시스템 전체의 안정성 확보

### ✔ 구현 내용

#### ↳ 장착/해제, 왼손 그립 위치, 장착 애니메이션/사운드 등 **무기 장착 시스템 통합**
- 코드: [무기 장착 시스템](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeapon.cpp#L140-L184)

    ```cpp
    bool ACWeapon::CanEquip()
    {
    	bool b = false;
    	b |= bEquipping;
    	b |= bReload;
    	b |= bFiring;
    	b |= State->IsInventoryMode() == true;
    
    	return !b;
    }
    
    void ACWeapon::Equip()
    {
    	bEquipping = true;
    	if (State == nullptr)
    		return;
    	if (Camera != nullptr)
    		Camera->EnableControlRotation();
    
    	State->SetEquipMode();
    	if (EquipSound != nullptr)
    		UGameplayStatics::SpawnSoundAtLocation(OwnerCharacter->GetWorld(), EquipSound, FVector::ZeroVector, FRotator::ZeroRotator);
    
    	if (EquipMontage == nullptr)
    	{
    		BeginEquip();
    		EndEquip();
    		return;
    	}
    
    	OwnerCharacter->PlayAnimMontage(EquipMontage, Equip_PlayRate);
    }
    
    void ACWeapon::BeginEquip()
    {
    	if (RightHandSokcetName.IsValid())
    		AttachToComponent(OwnerCharacter->GetMesh(), FAttachmentTransformRules(EAttachmentRule::KeepRelative, true), RightHandSokcetName);
    }
    
    void ACWeapon::EndEquip()
    {
    	bEquipping = false;
    	State->SetIdleMode();
    }
    ```
    
#### ↳ 카메라 FOV, 암 길이, 소켓 오프셋을 조절하는 **조준(Aim) 시스템 구현 (Timeline + Curve 기반)**
- 코드: [조준(Aim) 시스템 구현](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeapon.cpp#L421-L474)
    ```cpp
    bool ACWeapon::CanAim()
    {
    	bool b = false;
    	b |= bEquipping;
    	b |= bReload;
    	b |= bFiring;
    	b |= Weapon->IsKnifeMode() == true;
    	b |= Weapon->IsGrenadeMode() == true;
    	return !b;
    }
    
    void ACWeapon::BeginAim()
    {
    	ACPlayer* Player = Cast<ACPlayer>(OwnerCharacter);
    	if (!Player)
    		return;
    	bInAim = true;
    	if (BreathSoundComponent != nullptr && BreathSoundComponent->IsActive())
    		BreathSoundComponent->Stop();
    	if (BreathSound != nullptr)
    		BreathSoundComponent = UGameplayStatics::SpawnSoundAtLocation(GetWorld(), BreathSound, OwnerCharacter->GetActorLocation());
    	if (AimCurve != nullptr)
    	{
    		Timeline->PlayFromStart();
    		AimData.SetData(OwnerCharacter);
    		return;
    	}
    	AimData.SetDataByNoneCurve(OwnerCharacter);
    }
    
    void ACWeapon::EndAim()
    {
    	if (bInAim == false)
    		return;
    	bInAim = false;
    	if(BreathSoundComponent != nullptr && BreathSoundComponent->IsActive())
    		BreathSoundComponent->Stop();
    	if (BreathSound2 != nullptr)
    		BreathSoundComponent = UGameplayStatics::SpawnSoundAtLocation(GetWorld(), BreathSound2, OwnerCharacter->GetActorLocation());
    	if (AimCurve != nullptr)
    	{
    		Timeline->PlayFromStart();
    		BaseData.SetData(OwnerCharacter);
    		return;
    	}
    	BaseData.SetDataByNoneCurve(OwnerCharacter);
    }
    
    void ACWeapon::OnAiming(float Output)
    {
    	UCameraComponent* camera = Cast<UCameraComponent>(OwnerCharacter->GetComponentByClass(UCameraComponent::StaticClass()));
    	camera->FieldOfView = FMath::Lerp(AimData.FieldOfView, BaseData.FieldOfView, Output);
    }
    
    ```
    
#### ↳ 단발/자동 사격, 반동(Recoil), 카메라 셰이크 등 **발사 시스템 전반 처리**
- 코드: [**발사 시스템 전반 처리**](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeapon.cpp#L239-L349)     
    ```cpp
    void ACWeapon::BeginFire()
    {
    	bFiring = true;
    
    	State->SetActionMode();
    	if (bAutoFire == true)
    	{
    		GetWorld()->GetTimerManager().SetTimer(AutoFireHandle, this, &ACWeapon::OnFireing, AutoFireInterval, true, 0);
    		return;
    	}
    
    	OnFireing();
    }
    
    void ACWeapon::EndFire()
    {
    	if (bFiring == false)
    		return;
    	if (GetWorld()->GetTimerManager().IsTimerActive(AutoFireHandle))
    		GetWorld()->GetTimerManager().ClearTimer(AutoFireHandle);
    	State->SetIdleMode();
    	bFiring = false;
    }
    
    void ACWeapon::OnFireing()
    {
    	if (FireMontage != nullptr)
    		OwnerCharacter->PlayAnimMontage(FireMontage, FireRate);
    	UCameraComponent* camera = Cast<UCameraComponent>(OwnerCharacter->GetComponentByClass(UCameraComponent::StaticClass()));
    	FTransform transform{};
    	FVector direction{};
    	if (camera == nullptr)
    	{
    		transform = Mesh->GetSocketTransform("Muzzle_Bullet");
    		direction = transform.GetRotation().GetUpVector();
    	}
    	else
    	{
    		direction = camera->GetForwardVector();
    		transform = camera->GetComponentToWorld();
    	}
    
    	FVector start = transform.GetLocation() + direction;
    
    	direction = UKismetMathLibrary::RandomUnitVectorInConeInDegrees(direction, RecoilAngle);
    
    	FVector end = transform.GetLocation() + direction * HitDistance;
    
    	TArray<AActor*> ignores;
    	FHitResult hitResult;
    
    	UKismetSystemLibrary::LineTraceSingle(GetWorld(), start, end, ETraceTypeQuery::TraceTypeQuery1, false, ignores, Debug, hitResult, true, DebugColor);
    	if (hitResult.bBlockingHit == true)
    	{
    		if (HitDecal != nullptr)
    		{
    			FRotator rotator = hitResult.ImpactNormal.Rotation();
    			UDecalComponent* decal = UGameplayStatics::SpawnDecalAtLocation(GetWorld(), HitDecal, HitDecalSize, hitResult.Location, rotator, HitDecalLifeTime);
    			decal->SetFadeScreenSize(0);
    		}
    		if (HitParticle != nullptr)
    		{
    			FRotator rotator = UKismetMathLibrary::FindLookAtRotation(hitResult.Location, hitResult.TraceStart);
    			UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), HitParticle, hitResult.Location, rotator);
    		}
    	}
    
    	if (FlashParticle != nullptr)
    		UGameplayStatics::SpawnEmitterAttached(FlashParticle, Mesh, "Muzzle", FVector::ZeroVector, FRotator::ZeroRotator, EAttachLocation::KeepRelativeOffset);
    	if (EjectParticle != nullptr)
    		UGameplayStatics::SpawnEmitterAttached(EjectParticle, Mesh, "Eject", FVector::ZeroVector, FRotator::ZeroRotator, EAttachLocation::KeepRelativeOffset);
    	FVector muzzleLocation = Mesh->GetSocketLocation("Muzzle");
    	if (FireSound != nullptr)
    		UGameplayStatics::PlaySoundAtLocation(GetWorld(), FireSound, OwnerCharacter->GetMesh()->GetSocketLocation(L"pelvis"), OwnerCharacter->GetActorRotation(), 5);
    	if (CameraShak != nullptr)
    	{
    
    		APlayerController* controller = Cast<APlayerController>(OwnerCharacter->GetController());
    		if (controller != nullptr)
    		{
    			if (bInAim == true && AimCameraShak != nullptr)
    				controller->PlayerCameraManager->StartCameraShake(AimCameraShak,1,ECameraShakePlaySpace::UserDefined);
    			else
    				controller->PlayerCameraManager->StartCameraShake(CameraShak, 1, ECameraShakePlaySpace::UserDefined);
    		}
    	}
    
    	OwnerCharacter->AddControllerPitchInput(-RecoilRate * UKismetMathLibrary::RandomFloatInRange(0.8f, 1.2f));
    
    	if (BulletClass != nullptr)
    	{
    		FVector location = Mesh->GetSocketLocation("Muzzle_Bullet");
    		FActorSpawnParameters param;
    		param.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    
    		ACBullet* bullet = GetWorld()->SpawnActor<ACBullet>(BulletClass, location, direction.Rotation(), param);
    		bullet->OnHit.AddDynamic(this, &ACWeapon::OnBullet);
    		if (bullet != nullptr)
    			bullet->Shoot(direction);
    	}
    
    	if (CurrentMagazineCount > 1)
    		CurrentMagazineCount--;
    	else
    	{
    		CurrentMagazineCount--;
    		if (CanReload() == true)
    			Reload();
    	}
    }
    
    ```
    
#### ↳ 탄창 배출·장전·탄 수 관리 등 **탄창/재장전(Load/Unload) 시스템 구성**
- 코드: [**탄창/재장전(Load/Unload) 시스템 구성**](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeapon.cpp#L356-L419)	
    ```cpp
    bool ACWeapon::CanReload()
    {
    	bool b = false;
    	b |= bEquipping;
    	b |= bReload;
    	b |= CurrentMagazineCount >= MaxMagazineCount;
    	
    	return !b;
    }
    
    void ACWeapon::Reload()
    {
    	bReload = true;
    	EndAim();
    	EndFire();
    
    	if (ReloadMontage != nullptr)
    		OwnerCharacter->PlayAnimMontage(ReloadMontage, ReloadPlayRate);
    	
    	CurrentMagazineCount = MaxMagazineCount;
    
    	if (ReloadSound != nullptr)
    		UGameplayStatics::PlaySoundAtLocation(OwnerCharacter->GetWorld(), ReloadSound, FVector::ZeroVector, FRotator::ZeroRotator);
    
    }
    
    void ACWeapon::Eject_Magazine()
    {
    	if (MagazineBoneName.IsValid() == true)
    		Mesh->HideBoneByName(MagazineBoneName, PBO_None);
    	if (MagazineClass == nullptr)
    		return;
    
    	FTransform transform = Mesh->GetSocketTransform(MagazineBoneName);
    	ACMagazine* magazie = GetWorld()->SpawnActorDeferred<ACMagazine>(MagazineClass, transform, nullptr, nullptr, ESpawnActorCollisionHandlingMethod::AlwaysSpawn);
    	magazie->SetEject();
    	magazie->SetLifeSpan(5.0f);
    	magazie->FinishSpawning(transform);
    }
    
    void ACWeapon::Spawn_Magazine()
    {
    	if (MagazineClass == nullptr)
    		return;
    	FActorSpawnParameters param;
    	param.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    	Magazine = GetWorld()->SpawnActor<ACMagazine>(MagazineClass, param);
    	Magazine->AttachToComponent(OwnerCharacter->GetMesh(), FAttachmentTransformRules(EAttachmentRule::KeepRelative, true), MagazinSocketName);
    }
    
    void ACWeapon::Load_Magazine()
    {
    	CurrentMagazineCount = MaxMagazineCount;
    	if (MagazineBoneName.IsValid() == true)
    		Mesh->UnHideBoneByName(MagazineBoneName);
    
    	if (Magazine != nullptr)
    		Magazine->Destroy();
    }
    
    void ACWeapon::End_Magazine()
    {
    	bReload = false;
    }
    ```
    
#### ↳ HitResult 기반 데칼, 파티클, Bullet 콜백을 포함한 **피격 연출 및 데미지 흐름 처리**
- 코드: [**피격 연출 및 데미지 흐름 처리**](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeapon.cpp#L263-L325)
    ```cpp
    void ACWeapon::OnFireing()
    {
    	....
    	
    	FVector start = transform.GetLocation() + direction;
    
    	direction = UKismetMathLibrary::RandomUnitVectorInConeInDegrees(direction, RecoilAngle);
    
    	FVector end = transform.GetLocation() + direction * HitDistance;
    
    	TArray<AActor*> ignores;
    	FHitResult hitResult;
    
    	UKismetSystemLibrary::LineTraceSingle(GetWorld(), start, end, ETraceTypeQuery::TraceTypeQuery1, false, ignores, Debug, hitResult, true, DebugColor);
    	if (hitResult.bBlockingHit == true)
    	{
    		if (HitDecal != nullptr)
    		{
    			FRotator rotator = hitResult.ImpactNormal.Rotation();
    			UDecalComponent* decal = UGameplayStatics::SpawnDecalAtLocation(GetWorld(), HitDecal, HitDecalSize, hitResult.Location, rotator, HitDecalLifeTime);
    			decal->SetFadeScreenSize(0);
    		}
    		if (HitParticle != nullptr)
    		{
    			FRotator rotator = UKismetMathLibrary::FindLookAtRotation(hitResult.Location, hitResult.TraceStart);
    			UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), HitParticle, hitResult.Location, rotator);
    		}
    	}
    
    	if (FlashParticle != nullptr)
    		UGameplayStatics::SpawnEmitterAttached(FlashParticle, Mesh, "Muzzle", FVector::ZeroVector, FRotator::ZeroRotator, EAttachLocation::KeepRelativeOffset);
    	if (EjectParticle != nullptr)
    		UGameplayStatics::SpawnEmitterAttached(EjectParticle, Mesh, "Eject", FVector::ZeroVector, FRotator::ZeroRotator, EAttachLocation::KeepRelativeOffset);
    	FVector muzzleLocation = Mesh->GetSocketLocation("Muzzle");
    	if (FireSound != nullptr)
    		UGameplayStatics::PlaySoundAtLocation(GetWorld(), FireSound, OwnerCharacter->GetMesh()->GetSocketLocation(L"pelvis"), OwnerCharacter->GetActorRotation(), 5);
    	....
    }
    ```
    
#### 캐릭터 상태(StateComponent), 카메라(CameraComponent), 인벤토리와 연동하여
    
    → **무기–캐릭터–카메라의 통합 제어 구조 완성**
    

### ✔ 무기별 공격 방식

트리거 방식이 다른 무기들을 일관된 인터페이스(ACWeapon)를 유지하고 다형성을 활용한 로직 구현

#### ↳ **라이플 / 권총**
- [총 발사 로직](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeapon.cpp#L263-L354)
	- LineTrace를 활용하여 총을 쏘는 순간 조준한 방향, 충돌 위치를 판단하여 총알을 발사
	- 총알 발사 전 Bullet Class에 있는 델리게이트에 OnBullet 함수 바인딩
```cpp
void ACWeapon::OnFireing()
{
	if (FireMontage != nullptr)
		OwnerCharacter->PlayAnimMontage(FireMontage, FireRate);
	UCameraComponent* camera = Cast<UCameraComponent>(OwnerCharacter->GetComponentByClass(UCameraComponent::StaticClass()));
	FTransform transform{};
	FVector direction{};
	if (camera == nullptr)
	{
		transform = Mesh->GetSocketTransform("Muzzle_Bullet");
		direction = transform.GetRotation().GetUpVector();
	}
	else
	{
		direction = camera->GetForwardVector();
		transform = camera->GetComponentToWorld();
	}


	FVector start = transform.GetLocation() + direction;

	direction = UKismetMathLibrary::RandomUnitVectorInConeInDegrees(direction, RecoilAngle);

	FVector end = transform.GetLocation() + direction * HitDistance;

	TArray<AActor*> ignores;
	FHitResult hitResult;

	UKismetSystemLibrary::LineTraceSingle(GetWorld(), start, end, ETraceTypeQuery::TraceTypeQuery1, false, ignores, Debug, hitResult, true, DebugColor);
	
	....
	
	if (BulletClass != nullptr)
	{
		FVector location = Mesh->GetSocketLocation("Muzzle_Bullet");
		FActorSpawnParameters param;
		param.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

		ACBullet* bullet = GetWorld()->SpawnActor<ACBullet>(BulletClass, location, direction.Rotation(), param);
		bullet->OnHit.AddDynamic(this, &ACWeapon::OnBullet);
		if (bullet != nullptr)
			bullet->Shoot(direction);
	}

	....
}
```
- [총알 발사 및 충돌](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CBullet.cpp#L50-L71)
	-  ACWeapon Class에서 Shoot 함수 호출시 넘겨준 매개변수를 이용해 총알이 날아갈 방향 지정 후 Projectile에 설정된 Speed와 연산하여 총알 발사
	- 총알에 심어둔 Capsule Collition에 충돌시 OnComponentHit 함수를 호출하여 중복 충돌과 ACharacter 클래스가 아닌지 판단하여 데미지를 주도록 델리게이트 호출
```c++
void ACBullet::Shoot(const FVector& InDirection)
{
	SetLifeSpan(LifeTime);
	Projectile->Velocity = InDirection*Projectile->InitialSpeed;
	Projectile->SetActive(true);
}


void ACBullet::OnComponentHit(UPrimitiveComponent* HitComponent, AActor* OtherActor,
	UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
{
	for (AActor* actor : Ignores)
		if(actor == OtherActor)
			return;

	Capsule->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	ACharacter* character = Cast<ACharacter>(OtherActor);
	if (character != nullptr && OnHit.IsBound())
		OnHit.Broadcast(this, character);

	Destroy();
}
```
- [데미지 처리](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeapon.cpp#L475-L478)
	- 총알에서 델리게이트 호출시 ACWeapon::OnBullet 함수를 호출하여 데미지 처리
```cpp
void ACWeapon::OnBullet(AActor* InCauser, ACharacter* InOtherCharacter)
{
	HitDatas[0].SnedDamage(OwnerCharacter, InCauser, InOtherCharacter);
}
```


#### ↳ **칼(근접)** [근접 공격](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeapon_Knife.cpp#L63-L134)
- Sphere Collition를 이용해 애니메이션 실행시 Collition과 충돌 체크
- 충돌 감지시 데미지를 보내는 함수 호출
```cpp
void ACWeapon_Knife::OnComponentBeginOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor,
	UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	if(OtherActor == nullptr)
		return;
	APawn* other = Cast<APawn>(OtherActor);
	if(other == nullptr || other == OwnerCharacter)
		return;
	for (APawn* hitted : Hits)
		if(hitted == other)
			return;

	Hits.AddUnique(other);
	if(Hits.Num() <= 0)
		return;
	HitDatas[Index].SnedDamage(OwnerCharacter, this, other);
}

void ACWeapon_Knife::OnComponentEndOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor,
	UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
{
	Hits.Empty();
}

void ACWeapon_Knife::EnableCollision()
{
	Collisions[Index]->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
}

void ACWeapon_Knife::DisableCollision()
{
	for (UShapeComponent* shape : Collisions)
		shape->SetCollisionEnabled(ECollisionEnabled::NoCollision);
}
```
    
#### ↳ **수류탄 / 투척 무기**
    
    → Physics Simulation + ProjectileMovement로
    
    → 포물선 궤도 및 충돌 기반 폭발 판정 구현

```cpp
void ACWeapon_Throw::DonAction()
{
	if (State->IsIdleMode() == false)
		return;
	Super::DonAction();

	Data.DoAction(OwnerCharacter);
}

void ACWeapon_Throw::BeginAction()
{
	Super::BeginAction();
	GEngine->AddOnScreenDebugMessage(1, 2.0f, FColor::Green, FString::Printf(TEXT("Shoot")));
	ACGrenadesItem* greade = GetAttached();
	if (greade)
	{
		greade->DetachFromActor(FDetachmentTransformRules::KeepWorldTransform);
		UCameraComponent* camera = Cast<UCameraComponent>(OwnerCharacter->GetComponentByClass(UCameraComponent::StaticClass()));
		greade->Shoot(OwnerCharacter, camera->GetForwardVector());
		if(greade != nullptr)
			Grenades.Remove(greade);
	}
}

void ACWeapon_Throw::EndAction()
{
	Super::EndAction();
	GEngine->AddOnScreenDebugMessage(1, 2.0f, FColor::Green, FString::Printf(TEXT("Create")));
	Create();
}

void ACWeapon_Throw::Create()
{
	FVector location = OwnerCharacter->GetMesh()->GetSocketLocation("Grenade_Hand");
	FRotator rotation = OwnerCharacter->GetMesh()->GetSocketRotation("Greade_Hand");
	FActorSpawnParameters params;
	params.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
	params.Owner = OwnerCharacter;
	ACGrenadesItem* grenade = GetWorld()->SpawnActor<ACGrenadesItem>(GrenadesClass, location, rotation, params);
	if (grenade)
	{
		grenade->AttachToComponent(OwnerCharacter->GetMesh(), FAttachmentTransformRules::SnapToTargetNotIncludingScale, "Grenade_Hand");
		Grenades.Add(grenade);
	}

}

ACGrenadesItem* ACWeapon_Throw::GetAttached()
{
	for (ACGrenadesItem* greade : Grenades)
		if (greade != nullptr)
			return greade;

	return nullptr;
}

```

## 🧍‍♂️ **Character & Animation System**

### ✔ 구현 내용

## 💥 **Damage & Health System**

### ✔ 설계 의도

#### ↳ [Damage System](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Weapon/CWeaponStructures.cpp#L12-L24)
- SnedDamage 함수를 통해 
```cpp
void FHitData::SnedDamage(APawn* InAttacker, AActor* InAttackCauser, APawn* InOther)
{
	FActionDamageEvent e;
	e.HitData = this;
	InOther->TakeDamage(Power, e, InAttacker->GetController(), InAttackCauser);
}

void FHitData::PlayMontage(ACharacter* InOwner)
{
	if(HitMontage == nullptr)
		return;
	InOwner->PlayAnimMontage(HitMontage,PlayRate);
}
```


```cpp
void ACWeapon::OnBullet(AActor* InCauser, ACharacter* InOtherCharacter)
{
	HitDatas[0].SnedDamage(OwnerCharacter, InCauser, InOtherCharacter);
}
/////////////////////////////////////////////////////////////////////////////
void ACWeapon_Knife::OnComponentBeginOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor,
	UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	if(OtherActor == nullptr)
		return;
	APawn* other = Cast<APawn>(OtherActor);
	if(other == nullptr || other == OwnerCharacter)
		return;
	for (APawn* hitted : Hits)
		if(hitted == other)
			return;

	Hits.AddUnique(other);
	if(Hits.Num() <= 0)
		return;
	HitDatas[Index].SnedDamage(OwnerCharacter, this, other);
}
```


# **Troubleshooting**

### 1) 🎯 몽타주가 중첩 재생되면 이후 몽타주가 재생되지 않는 문제
- 문제: 몽타주가 실행되는 중, 피격을 당하면 피격 몽타주 재생 후 다른 행동을 하지 못하는 문제 발생
- 원인: 몽타주의 Notify로 설정한 Begin, End 함수중 Begin은 실행되었지만 End가 실행되지 못해 StateComponent의 상태가 변화하지 못해 생기는 문제
- 해결: CCharacter 클래스에서 UAnimInstance안에 OnMontageEnded 델리게이트를 연결하여 몽타주가 중간에 끊겼을 때 강제로 Begin, End 함수를 호출하도록 하여 해결
[ACCharacter](https://github.com/GyungSikHan/1st-Team14-CH3-Project/blob/main/Source/Start/Private/CCharacter.cpp#L106-L140)
```cpp
void ACCharacter::BeginPlay()
{
    Super::BeginPlay();

	....

	UAnimInstance* instance = GetMesh()->GetAnimInstance();
    if (instance != nullptr)
    	instance->OnMontageEnded.AddDynamic(this, &ACCharacter::HandleAnyMontageEnded);
}

void ACCharacter::HandleAnyMontageEnded(UAnimMontage* Montage, bool bInterrupted)
{
    if(Montage == nullptr)
        return;
    if(Montage->GetName() == "Equip_Rifle_Standing_Montage")
    {
        WeaponComponent->Begin_Equip();
        WeaponComponent->End_Equip();
        return;
    }
    if(Montage->GetName() == "Reload_Rifle_Hip_Montage")
    {
	    WeaponComponent->Load_Magazine();
	    WeaponComponent->End_Magazine();
	    return;
    }
    if(Montage->GetName() == "Equip_Pistol_Standing_Montage")
    {
        WeaponComponent->Begin_Equip();
        WeaponComponent->End_Equip();
    }
    if(Montage->GetName() == "Reload_Pistol_Montage")
    {
        WeaponComponent->Load_Magazine();
        WeaponComponent->End_Magazine();
    }
    if(Montage->GetName() == "Grenade_Throw_Montage")
    {
        WeaponComponent->End_DoAction();
    }
    if(Montage->GetName() == "Fist_Attack_1_Montage" || Montage->GetName() =="Fist_Attack_2_Montage")
    {
        WeaponComponent->End_DoAction();
    }
}

```

### 2) 🎯 수류탄이 자연스럽게 투척되지 않는 문제
- 문제: ProjectileMovementComponent를 사용하여 포물선을 그리며 날아가지 않고 땅에 바로 떨어지거나 앞으로만 날아가는 문제 발생
- 원인: GravityScale만 조절하여 구현하다보니 자연스럽지 않고 부자연스럽게 날아가게 됨
- 해결1: Mesh의 물리 시뮬레이션을 활성화 하여 위치 이동뿐만 아니라 실제 물리 법칙에 따르게 만들면서 AddImpulse() 함수를 활용하여 순간적인 힘을 가하도록 구현
- 해결2: AddTorqueInRadians() 함수를 통해 특정 범위 내에서 수류탄이 회전하도록 구현

```cpp
void ACGrenadesItem::Shoot(const ACharacter* OwnerCharacter, const FVector& InDirection)
{
	// 충돌 설정
	SkeletalMesh->SetCollisionProfileName("PhysicsActor");

	// 투사체 속도 및 방향 설정
	FVector LaunchVelocity = InDirection * Projectile->InitialSpeed;

	// 충돌체에 물리적 힘 적용
	SkeletalMesh->SetSimulatePhysics(true);
	SkeletalMesh->AddImpulse(LaunchVelocity, NAME_None, true);

	// 회전 추가 (더 자연스러운 던지기 효과)
	FVector RandomTorque = FVector(
		FMath::RandRange(-500.0f, 500.0f),
		FMath::RandRange(-500.0f, 500.0f),
		FMath::RandRange(-500.0f, 500.0f)
	);
	SkeletalMesh->AddTorqueInRadians(RandomTorque, NAME_None, true);

	// 소유자 충돌 무시
	SkeletalMesh->IgnoreActorWhenMoving(Owner, true);

	// 투사체 활성화
	Projectile->SetActive(true);

	// 폭발 타이머 설정
	GetWorld()->GetTimerManager().SetTimer(
		ExplosiveTimerHandle,
		this,
		&ACGrenadesItem::Explode,
		LifeTime,
		false
	);
}
```

### 3) 🎯 SymBio의 스플라인 공격에 충돌이 적용되지 않는 문제
- 문제: Skill 구현을 맡은 팀원이 에셋 분석 중 스플라인의 충돌 부분에서 Actor를 가져오지 못하여 데미지를 주지 못함
- 원인: 스플라인의 끝부분에 충돌한 위치는 가져오지만, Actor를 가져오지 못함
- 해결: 스플라인을 그리기 전, LineTrace를 그리는 로직이 있어 그곳에 있는 HitResult 중 HitActor를 가져와 CCharacter로 케스팅 후 ApplyDamage를 통해 데미지를 넘기도록 결
 <table>
    <tr>
        <td align="center">
             <img src="image-4.png" width="600"><br>
            <em>Blueprint</em>
		</td>
		        <td align="center">
             <img src="image-3.png" width="400"><br>
            <em>Skill 사용</em>
		</td>
</table>

# **Retrospective (느낀점)**
