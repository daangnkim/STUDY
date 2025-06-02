### 필드를 required하게 만들기

다음은 필드를 required하게 만들지 않는다.

```typescript
const schema = z.object({
	title: z.string({ required_error: '필수 입력이에요' })
})
```

왜냐하면 다음 코드에서는 title을 입력하지 않아도 입력된 것처럼 취급한다.

```typescript
const schema = z.object({
	title: z.string({ required_error: '필수 입력이에요' })
})


const { register, control, setValue, handleSubmit, watch, formState } =
	useForm<z.infer<typeof AddServiceSchema>>({
	resolver: zodResolver(AddServiceSchema),
	defaultValues: {
		title: '',
	},
});

```

그러므로 `min(1, { message: '필수 입력 필드입니다' })`를 사용해야한다.

### input 요소에 초기값을 할당하지 않은 상태에서 입력을 하면 에러가 발생한다.
 [javascript - A component is changing an uncontrolled input of type text to be controlled error in ReactJS - Stack Overflow](https://stackoverflow.com/questions/47012169/a-component-is-changing-an-uncontrolled-input-of-type-text-to-be-controlled-erro)

### Expected Number, Received String
schema가 number인 필드에 대해서 Expected number, received string 에러가 발생하는 이슈
[colinhacks/zod: TypeScript-first schema validation with static type inference](https://github.com/colinhacks/zod?tab=readme-ov-file#coercion-for-primitives)
[ZodResolver error: "Expected number, received string" · Issue #73 · react-hook-form/resolvers](https://github.com/react-hook-form/resolvers/issues/73)

### radio 버튼에 react-hook-form 사용하기
[reactjs - Radio buttons with react-hook-form - Stack Overflow](https://stackoverflow.com/questions/67626696/radio-buttons-with-react-hook-form)

### 하나의 필드가 optional한지 required한지를 다른 필드에 의존하게 만들기


### optional vs nullable vs nullish의 차이
[zod optional/nullable/nullish differences](https://gist.github.com/ciiqr/ee19e9ff3bb603f8c42b00f5ad8c551e)


### schema가 number인 필드에 대해서 Expected number, received string 에러가 발생하는 이슈

[colinhacks/zod: TypeScript-first schema validation with static type inference](https://github.com/colinhacks/zod?tab=readme-ov-file#coercion-for-primitives)
[ZodResolver error: "Expected number, received string" · Issue #73 · react-hook-form/resolvers](https://github.com/react-hook-form/resolvers/issues/73)

### watch vs useWatch

[React Hook Form: Understanding watch vs useWatch - DEV Community](https://dev.to/kcsujeet/react-hook-form-understanding-watch-vs-usewatch-l54)

### 초기값으로 ""을 할당하는 것의 문제점

1. onChange 이벤트가 발생하면 e.target.value에 접근했을 때 option 태그의 value가 찍힌다.
2. option 태그는 value를 할당하지 않으면 textContent가 value로 사용된다.
3. 2에 의해서 option 태그의 초기 option에 대한 value를 ""를 설정해야한다.
4. value를 ""로 설정하게 만들면, url에 query param을 설정할 때 ""를 제외하고 작성해야한다.
5. value를 ""로 설정하게 만들면, zod schema에서 ""에 대한 허용을 하나하나 적용해줘야한다.
6. value를 ""로 설정하게 만들면, 