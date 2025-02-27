---
title: react-hook-form을 활용한 폼 상태 관리와 UI 로직 분리
date: 2024-11-02 12:15:00 +09:00
categories: [react-hook-form]
tags: [react-hook-form, 컴파운드 패턴]
---

> 이 글은 `react-hook-form: "^7.54.2"` 버전을 기반으로 작성되었습니다.

## 기존 문제점과 개선 목표

기존에는 폼 상태와 UI 로직이 뒤섞여 코드 가독성과 유지보수성이 떨어졌습니다. <br/>
이를 해결하기 위해 SRP(단일 책임 원칙)를 적용하며 `react-hook-form`을 도입하고 컴포넌트를 최적화해 다음 목표를 세웠습니다.

- 폼 상태를 외부로 분리해 UI를 단순화
- 공통 컴포넌트와 컴파운드 패턴으로 재사용성 강화
- 직관적인 코드 구조로 유지보수성 개선

---

## 개선 과정

### 1. react-hook-form으로 상태 관리 간소화

폼의 상태 관리와 UI 로직이 뒤섞여 가독성과 유지보수성이 저하되는 문제를 해결하고자, SRP를 바탕으로 역할을 분리했습니다. 
상태 관리는 `react-hook-form`에 위임하고, `Form` 컴포넌트는 UI 렌더링만 담당하는 Dumb Component로 설계했습니다.

```tsx
export const Form = ({ methods, children, ...props }) => {
  return (
    <FormProvider {...methods}>
      <form {...props}>{children}</form>
    </FormProvider>
  );
};

// 사용 예시
const methods = useForm({
  defaultValues: { email: '', password: '' }
});

const { handleSubmit } = methods;

<Form methods={methods} onSubmit={handleSubmit(onSubmit)}>
  <Form.Text name="email" placeholder="이메일" />
  <Form.Password name="password" placeholder="비밀번호" />
  <button type="submit">로그인</button>
</Form>
```

### 2. 컴파운드 패턴으로 입력 필드 개선

폼 안에서 `input`과 `label` 등 반복적인 구조가 발견됐고, 일부 폼은 스타일링 측면에서 `label`과 `input`의 관계가 달라지는 경우가 있어 이를 어떻게 처리할지 고민했습니다.
SRP를 적용해 각 컴포넌트가 단일 책임을 지도록, 컴파운드 패턴을 도입해 입력 필드를 공통화하고 유연성을 확보했습니다.

```tsx
const FormText = ({ name, ...props }) => {
  const { register, formState: { errors } } = useFormContext()
  return (
    <div>
      <TextInput {...register(name)} {...props} error={Boolean(errors[name])} />
      {errors[name] && <span>{errors[name].message}</span>}
    </div>
  )
}

// 중간 생략 ...

Form.Text = FormText
Form.Password = FormPassword
// 추가 지원: TextArea, Checkbox, Radio 등이 가능
```

- 컴파운드 패턴 선택 이유: 폼 내 반복 구조를 모듈화하면서도, 스타일링 차이(예: label 위치나 크기)에 유연하게 대응할 수 있어 적합했습니다.
- `useFormContext` 이점: `react-hook-form`의 컨텍스트를 활용해 부모와 하위 컴포넌트 간 상태를 공유하며, Form.Text 안에서 errors 같은 상태를 바로 사용할 수 있었습니다. 이는 SRP를 지키며 상태 관리 책임을 `react-hook-form`에 집중시킨 결과입니다.
- 장점: 각 도메인의 폼 필드가 일관된 방식으로 구성돼, 상태를 한 곳에서 관리하면서도 다양한 폼을 쉽게 확장할 수 있었습니다.

### 3. Controller로 동적 입력 처리 강화

SRP를 적용해 `FormRadio`가 상태 관리와 UI 렌더링을 분리하며 단일 책임을 갖도록 했습니다.

```tsx
const FormRadio = ({ name, rules, options, ...props }) => {
  const { control } = useFormContext()
  return (
    <Controller
      name={name}
      control={control}
      rules={rules}
      render={({ field }) => (
        <>
          {options.map(option => (
            <RadioInput
              key={option.value}
              label={option.label}
              value={option.value}
              checked={field.value === option.value}
              onChange={() => field.onChange(option.value)}
              {...props}
            />
          ))}
        </>
      )}
    />
  )
}
```
- Controller 활용: `control`을 통해 `react-hook-form`의 상태를 동적으로 관리하며, 옵션이 많은 `RadioInput` 같은 입력을 유연하게 처리했습니다.
- 이점: 복잡한 입력(예: 라디오 그룹)을 컴포넌트 단위로 캡슐화해 유지보수성과 확장성을 높였습니다.

---

## 성과

- 작업 효율성 향상: 공통 컴포넌트로 반복 작업을 줄여 개발 속도 개선
- 가독성 및 유지보수성: SRP 기반 상태와 UI 분리로 코드 구조를 명확히 정리
- 확장성: `Controller`와 컴파운드 패턴으로 Text, Password 외에도 Checkbox, Radio 등 다양한 폼을 손쉽게 구성 가능

`react-hook-form` 도입과 SRP를 적용한 컴포넌트 설계를 통해, 팀원들이 사전 지식 없이도 간단한 props로 폼을 구성할 수 있는 환경을 만들었고, 프로젝트 효율성과 협업 생산성을 크게 끌어올렸습니다.
