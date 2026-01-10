# 프로젝트 소개

# 📌 프로젝트 개요

"**SymBio**"는 언리얼 엔진 기반으로 제작된 1인칭 FPS 게임으로, 극한의 생존을 위해 인간과 SymBio가 정신적·신체적으로 융합된 세계관을 담고 있습니다. 플레이어는 제한된 자원과 능력 속에서 전투와 생존을 동시에 수행해야 합니다.

# 🎮 게임 개발
> 
> - **인원**: 4인 개발  
> - **기간**: 25.02.17 ~ 25.03.07
> - **목적**: 언리얼 엔진의 **Gameplay Framework**와 **Component 기반 구조**를 이해하고, C++과 Blueprint 기반으로 FPS 게임의 핵심 시스템(무기, 캐릭터, 데미지)을 직접 구현하는 경험을 목표로 함.
> - **기술**: - C++, Unreal Engine 5.5, Blueprint, Git, Slack, Rider/Visual Studio
</aside>

## 🖼 In-Game Screenshot

![alt text](image-1.png)
![alt text](image-2.png)
---

# **Core System Implementation**

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

ACWeapon 베이스에서 **공격 판정 방식을 타입별로 분리**하여 구현했습니다.

- **라이플 / 권총**
    
    → LineTrace 기반 **Hitscan**으로 
    
- **칼(근접)**
    
    → SphereTrace를 이용해 근접 타격 범위 기반 판정
    
- **수류탄 / 투척 무기**
    
    → Physics Simulation + ProjectileMovement로
    
    → 포물선 궤도 및 충돌 기반 폭발 판정 구현
    

각 무기 타입은 같은 인터페이스로 동작하지만,

**판정 방식만 독립적으로 확장할 수 있는 구조**로 설계되어 유지보수가 용이합니다.

## 🧍‍♂️ **Character & Animation System**

### ✔ 구현 내용

## 💥 **Damage & Health System**

### ✔ 설계 의도

## ⚙ **Component Architecture**

### ✔ 구현 내용

# **Troubleshooting**

### 1) 🎯 몽타주가 중첩 재생되면 이후 몽타주가 재생되지 않는 문제

### 2) 🎯 수류탄이 자연스럽게 투척되지 않는 문제

### 3) 🎯 SymBio의 스플라인 공격에 충돌이 적용되지 않는 문제

# **Retrospective (느낀점)**

# 게임 플레이 영상

<p align="center">
  <a href="https://youtu.be/tJcozATuAgQ">
    <img src="image.png" width="1000">
  </a>
</p>