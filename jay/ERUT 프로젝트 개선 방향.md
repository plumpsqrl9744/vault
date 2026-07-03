
### 1. 빌드 및 CI/CD 성능 최적화

- **문제 상황:** 빌드 타임이 과도하게 소요되어 생산성 저하 및 CI/CD 병목 발생.
	- 기존 빌드 진행 시 증분 빌드를 통해 캐싱을 활용해 패키지 install 시간을 감소시켜 평균적으로 20~30% 감소 시간을 감축했지만, 여전히 패키지 install 에서 시간이 대략 1분 내외로 소모됨 
    
- **해결 방안:** * **패키지 관리:** `pnpm`을 도입하여 효율적인 의존성 관리를 수행합니다.
    
    - **캐시 최적화:** Jenkins 빌드 서버의 경로(예: `/var/lib/jenkins/.pnpm-store`)를 볼륨으로 마운트하여, 의존성 설치 캐시를 재사용함으로써 네트워크 비용과 빌드 시간을 획기적으로 단축합니다.
### 2. 컴포넌트 로직 및 관심사 분리

- **문제 상황:** 컴포넌트 내부의 복잡한 `if` 분기 처리로 인해 가독성이 저하되고 기능 확장 시 컴포넌트 오염 발생.
    
- **해결 방안 예시:**
    
    - **관심사 분리:** 드롭다운 컴포넌트의 역할을 '로직(Container)'과 '뷰(Presentation)'로 엄격히 분리합니다.
        
    - **구조 개선:** 드롭다운은 상태와 변경 로직만 제어하고, 옵션 리스트는 데이터를 전달받아 `map` 로직으로 렌더링하는 순수 UI 컴포넌트로 분리합니다. 이를 통해 복잡한 분기문을 제거하고 결합도를 낮춥니다.

**❌ BEFORE — `ams/web/src/shared/components/dropdown/Dropdown.tsx`**
> 한 컴포넌트가 상태·필터·키보드·모드분기·렌더를 전부 소유(약 400줄). 파생 상태를 useEffect로 동기화하고 `mode`/키보드/값결정마다 분기.

```tsx
export default function Dropdown({ mode, options, value, onChange, allowCustomValue }) {
  const [isOpen, setIsOpen] = useState(false);
  const [searchValue, setSearchValue] = useState('');
  const [selectedValue, setSelectedValue] = useState<Option | null>(null);
  const [highlightedIndex, setHighlightedIndex] = useState(-1);

  // 파생 상태를 useEffect로 동기화 → source of truth 이중화(안티패턴)
  useEffect(() => {
    const matched = options.find((o) => String(o.value) === String(value));
    if (matched) {
      setSelectedValue(matched);
      if (mode === 'search') setSearchValue(matched.label);
    } else if (allowCustomValue && mode === 'search') {
      setSelectedValue(null); setSearchValue(String(value));
    } else {
      setSelectedValue(null); if (mode === 'search') setSearchValue('');
    }
  }, [value, options, mode, allowCustomValue]);

  const handleKeyDown = (e) => {
    switch (e.key) {                              // 키보드 분기
      case 'ArrowDown': /* ... */ break;
      case 'ArrowUp':   /* ... */ break;
      case 'Enter':     /* ... */ break;
      case 'Escape':    /* ... */ break;
    }
  };

  const handleSearchChange = (e) => {
    const exact = options.find((o) => o.label === e.target.value);
    if (exact && !exact.disabled) { /* ... */ }   // 값 결정 분기
    else if (allowCustomValue)    { /* ... */ }
    else if (selectedValue)       { /* ... */ }
  };

  const filteredOptions =
    mode === 'search'
      ? options.filter((o) => o.label.toLowerCase().includes(searchValue.toLowerCase()))
      : options;

  return (
    <div>
      {mode === 'select' && <button>{selectedValue?.label || placeholder}</button>}  {/* 모드 분기 */}
      {mode === 'search' && <div><input value={searchValue} onChange={handleSearchChange} /></div>} {/* 모드 분기 */}
      {isOpen && <DropdownMenu options={filteredOptions} />}
    </div>
  );
}
```

**✅ AFTER — `exam-components` (Container / Registry / Presentation / Adapter)**
> Container는 상태·변경만, 렌더러는 레지스트리 lookup으로 dispatch → `if` 분기 제거. 뷰는 `map`만 하는 순수 컴포넌트.

```tsx
// 1) 로직(Container): 상태·변경만 제어, 렌더러는 레지스트리로 dispatch
// base/select/Select.tsx
export const Select = ({ type, options, value, onChange }: SelectProps) => {
  const [open, setOpen] = useState(false);
  const ctx = {
    value,
    isSelected: (v) => value.includes(v),
    select: (v) => { onChange([v]); setOpen(false); },                                  // 단일: 교체
    toggle: (v) => onChange(value.includes(v) ? value.filter((x) => x !== v) : [...value, v]), // 다중: 토글
  };
  const Renderer = OPTION_RENDERERS[type];   // ← 유일한 분기 = lookup 한 줄 (switch/if 없음)
  return (
    <div className={styles.root}>
      <button onClick={() => setOpen((o) => !o)}>{/* trigger */}</button>
      {open && (
        <div className={styles.menu}>
          <SelectContext.Provider value={ctx}>
            <Renderer options={options} />
          </SelectContext.Provider>
        </div>
      )}
    </div>
  );
};

// 2) 레지스트리: 타입 추가 = 렌더러 파일 + 여기 한 줄 (Container 본문 불변 → OCP)
// base/select/option/index.ts
export const OPTION_RENDERERS = {
  select: PlainList,      // 단일 선택
  checkbox: CheckboxList, // 다중 선택
} as const;
export type SelectType = keyof typeof OPTION_RENDERERS;

// 3) 뷰(Presentation): 데이터 받아 map만, 자기 선택 정책(toggle)만 소유
// base/select/option/CheckboxList.tsx
export const CheckboxList = ({ options }: OptionRendererProps) => {
  const { isSelected, toggle } = useContext(SelectContext);
  return options.map((o) => (
    <button key={o.value} onClick={() => toggle(o.value)}>
      {isSelected(o.value) ? '[v]' : '[ ]'} {o.label}
    </button>
  ));
};

// 4) 폼 결합은 어댑터 한 겹에만 (RHF ↔ value/onChange, context는 모름)
// form/FormSelect.tsx
export const FormSelect = ({ name, type, options }) => {
  const { control } = useFormContext();
  const { field } = useController({ name, control });
  const multiple = MULTI_TYPES.includes(type);
  const value = multiple ? (field.value ?? []) : field.value != null ? [field.value] : [];
  return (
    <Select
      type={type}
      options={options}
      value={value}
      onChange={(next) => field.onChange(multiple ? next : next[0] ?? null)}
    />
  );
};
```



        

### 3. CSS DX 개선 및 스타일링 컨벤션

- **문제 상황:** 컴포넌트/페이지별 무분별한 로컬 SCSS 파일 생성으로 인한 스타일 파편화 및 관리 비용 증가.
    
- **해결 방안:**
    
    - **기술 스택:** Tailwind CSS 도입을 권장하되, SCSS 유지 시에는 엄격한 컨벤션을 적용합니다.
        
    - **공통 스타일:** 디자인 시스템(컬러, 폰트, 간격 등) 토큰과 공통 Reset 코드만 전역 파일로 관리합니다.
        
    - **로컬 스타일:** `ComponentName.module.scss` 명명 규칙 준수 및 BEM 방법론을 도입하여, 재사용이 불가능한 고유 영역에 한해서만 격리된 스타일을 정의합니다.
        

### 4. FSD(Feature-Sliced Design) 구조 정착 (3단계 방어 체계)

- **문제 상황:** FSD 아키텍처에 대한 이해 부족으로 디렉토리 구조가 파편화되고 의존성 규칙이 준수되지 않음.
    
- **해결 방안 (3단계 중첩 적용):**
    
    - **[단계 1] Scaffolding CLI:** `hygen`/`plop`을 통해 표준 FSD 구조에 맞춘 파일 템플릿을 생성하여 휴먼 에러 원천 차단.
        
    - **[단계 2] ESLint:** `eslint-plugin-import`로 레이어 간 import 경로를 즉각적으로 제한하여 IDE 레벨에서 규칙 준수 유도.
        
    - **[단계 3] Dependency Cruiser:** CI 파이프라인의 최종 문지기로서, 아키텍처 규칙 위반 시 빌드를 차단하여 물리적 강제력 확보.
        

**[종합 요약 로드맵]**

|**단계**|**핵심 과제**|**해결 수단**|
|---|---|---|
|**1. 빌드**|속도/파이프라인 최적화|pnpm 스토어 캐싱, 증분 빌드|
|**2. 로직**|관심사 분리 (분기 제거)|Container/Presentation 패턴|
|**3. 스타일**|DX 개선 및 파편화 방지|Tailwind/공통 토큰/모듈 단위 격리|
|**4. 구조**|FSD 아키텍처 강제|CLI(생성) -> ESLint(검사) -> Dep-Cruiser(빌드 차단)|
