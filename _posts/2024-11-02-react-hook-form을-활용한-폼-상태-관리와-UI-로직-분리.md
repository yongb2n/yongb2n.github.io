---
title: react-hook-form을 활용한 폼 상태 관리와 UI 로직 분리
date: 2024-11-02 12:15:00 +09:00
categories: [react-hook-form]
tags: [react-hook-form, 컴파운드 패턴]
---

## 기존 문제점과 개선 목표

기존에는 폼 상태와 UI 로직이 뒤섞여 코드 가독성과 유지보수성이 떨어졌습니다. <br/>
이를 개선하기 위해 React Hook Form 도입과 컴포넌트 최적화를 통해 다음 목표를 세웠습니다.

- 폼 상태를 외부로 분리해 UI를 단순화
- 공통 컴포넌트와 컴파운드 패턴으로 재사용성 강화
- 직관적인 코드 구조로 유지보수성 개선

---

## 개선 과정

### 1. react-hook-form으로 상태 관리 간소화

폼 상태를 `react-hook-form`에 위임해 UI 컴포넌트를 단순화하고, 중앙화된 상태 관리를 구현했습니다.

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

### 2. 입력 필드 컴포넌트 공통화

반복되는 입력 필드를 `Text`, `Password` 등으로 분리해 재사용 가능한 범용 컴포넌트로 설계했습니다.

```tsx
const FormText = ({ name, ...props }) => {
  const { register, formState: { errors } } = useFormContext();
  return (
    <div>
      <TextInput {...register(name)} {...props} error={Boolean(errors[name])} />
      {errors[name] && <span>{errors[name].message}</span>}
    </div>
  )
}
```

### 3. 컴파운드 패턴 적용

Form 컴포넌트를 유기적으로 구성해 일관성과 확장성을 확보했습니다. `email`, `password` 외에도 다양한 입력 필드를 지원합니다.

```tsx
Form.Text = FormText
Form.Password = FormPassword

// 추가 지원으로는 TextArea, Checkbox, Radio, TagInput, File 등이 있습니다.
```

이 방식으로 **각 도메인의 폼 필드가 일관된 방식으로 구성**될 수 있도록 하여, **폼 상태를 한 곳에서 관리하면서도 다양한 폼을 쉽게 생성**할 수 있도록 했습니다.

### 4. 파일 업로드 확장

기존 구조를 활용해 파일 업로드 기능도 유연하게 추가했습니다.

```tsx
const FormFile = ({ name }) => {
  const { control, watch, setValue } = useFormContext()
  const files = watch(name) || []
  return (
    <Controller
      name={name}
      control={control}
      render={({ field }) => (
        <FileUploadUI files={files} onChange={(newFiles) => setValue(name, newFiles)} />
      )}
    />
  )
}

// Checkbox, Radio 등 다른 입력 유형도 비슷한 방식으로 확장 가능
```

---

## 성과

- 작업 효율성 향상: 공통 컴포넌트로 반복 작업을 줄여 개발 속도 개선
- 가독성 및 유지보수성: 상태와 UI 분리로 코드 구조를 명확히 정리
- 확장성: 컴파운드 패턴으로 Text, Password 외에도 Checkbox, File 등 다양한 폼을 손쉽게 구성 가능

react-hook-form의 도입과 컴포넌트 최적화를 통해 **팀원들이 사전 지식 없이도 간단한 props만으로 폼을 구성할 수 있는 환경**을 제공할 수 있었고, 프로젝트 효율성과 협업 생산성을 눈에 띄게 끌어올렸습니다.
