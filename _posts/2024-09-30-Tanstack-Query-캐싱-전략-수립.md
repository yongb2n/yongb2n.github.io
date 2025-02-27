---
title: Tanstack Query로 권한 제어 강화된 그룹 관리
date: 2025-09-29 12:15:00 +09:00
categories: [권한 제어]
tags: [Tanstack Query, 권한 제어]
---
## 배경과 문제 상황
서비스에서 사용자가 그룹을 생성하고 관리할 수 있게 설계했지만, 권한 제어가 없어 비관리자가 생성, 수정, 삭제 작업을 수행할 수 있는 보안 취약점이 있었습니다. 
이를 해결하기 위해 `isAdmin` 값으로 관리자 권한을 판단하고, UI와 로직을 분리해 보안과 유지보수성을 동시에 강화하고자 했습니다.

## 개선 목표
- `isAdmin`으로 무단 접근 차단 및 권한 기반 작업 제한  
- TanStack Query로 서버 데이터 캐싱 최적화  
- SRP 적용으로 코드 구조 간소화

## 개발 과정
### 1. 문제 분석과 설계 접근
초기에는 버튼 클릭 시마다 서버에 권한을 확인하는 방식이었지만, 성능 저하와 네트워크 부하가 생길 것으로 판단했습니다. <br/>
이를 해결하기 위해 TanStack Query로 그룹과 사용자 데이터를 캐싱하고, 클라이언트에서 `isAdmin`을 계산해 UI를 동적으로 조정하는 방법을 선택했습니다. <br/>
SRP를 적용해 권한 로직과 UI 렌더링을 분리했습니다. 

- **접근법**: 서버에서 가져온 데이터를 실시간으로 활용해 권한을 확인하고, 조건부 렌더링으로 보안 강화

### 2. TanStack Query로 데이터 캐싱 및 권한 계산
그룹과 사용자 데이터를 효율적으로 가져와 캐싱하고, 이를 기반으로 `isAdmin`을 계산했습니다.

**쿼리 정의**
```tsx
// src/queries/config.ts
export const queryOptions = {
  staleTime: 1000 * 60 * 5, // 5분 
  gcTime: 1000 * 60 * 10,   // 10분
};


// src/queries/group/index.ts
export const useGroupsQuery = (id: string) => {
  return useQuery({
    queryKey: ['groups', id],
    queryFn: () => getGroups(id),
    staleTime: 1000 * 60 * 60, // 1시간
    gcTime: 1000 * 60 * 60 * 24, // 24시간    
    retry: 1,
  });
};

// src/queries/user/index.ts
export const useUsersQuery = (accessToken: string | null) => {
  return useQuery({
    queryKey: ['user'],
    queryFn: getUsers,
    enabled: !!accessToken,
    staleTime: queryOptions.staleTime,
    gcTime: queryOptions.gcTime,
    retry: 1,
  });
};
```

- 설정: `staleTime`과 `gcTime`은 기본값(5분/10분) 사용. 그룹 데이터는 정적이라 더 길게, 사용자 데이터는 적당히 동적이라 적합하다고 판단
- 캐싱 이점: 권한 확인에 필요한 데이터가 캐시에서 빠르게 로드돼 성능 향상
- 권한 계산 로직:
```tsx
const isAdmin = groupData && userData
  ? groupData.members.some(
      (member) => member.userId === userData.data.id && member.role === 'ADMIN'
    )
  : false;
```

- 어떻게: `groupData.members` 배열을 순회하며 현재 사용자(`userData.data.id`)와 역할(`role === 'ADMIN'`)을 비교
- 가정: `userData.data.id`는 API 응답의 사용자 ID 필드

### 3. 권한 기반 UI 동적 제어
`isAdmin` 값을 기준으로 UI를 동적으로 조정하며, SRP를 적용해 권한 로직과 렌더링을 분리했습니다.

- 조건부 렌더링:
```tsx
{isAdmin && (
  <button
    onClick={handleOpenInviteModal}
    ...
  >
    새로운 멤버 초대하기
  </button>
)}
```

- 확장: 수정/삭제 버튼도 동일 로직 적용
```tsx
 {isAdmin && (
  <div>
    <button onClick={handleEditNotificationModal}>
      <Image src={editIcon} alt="수정" width={16} height={16} />
    </button>
    <button onClick={handleDeleteNotificationModal}>
      <Image src={deleteIcon} alt="삭제" width={16} height={16} />
    </button>
  </div>
)}
```

### 4. 캐시 무효화와 최신성 유지
그룹 데이터 변경 시 캐시를 갱신해 권한 상태를 최신화했습니다.

- Mutation 예시:
```tsx
export const useCreateGroupMutation = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ teamId, name, image }) => createGroup(teamId, { name, image }),
    onSuccess: async () => {
      await queryClient.invalidateQueries({ queryKey: ['groups'] });
      await queryClient.invalidateQueries({ queryKey: ['user'] });
    },
  });
};
```

- 어떻게: 그룹 생성 후 `invalidateQueries`로 캐시 무효화, 서버에서 최신 `groupData`와 `userData` 재조회
- 이점: 비관리자가 캐시된 권한으로 작업 시도 방지, 일관성 유지

## 결과
SRP와 캐시 전략으로 코드 수정 용이성을 증가시키며, `isAdmin`을 기반으로 권한 제어 로직을 구현하여 특정 작업을 수행할 수 있는 사용자와 없는 사용자를 명확하게 구분했습니다.
