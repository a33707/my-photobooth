# 가족 포토부스 코드 분석

## 개요
`index.html` 파일은 웹 기반 가족 포토부스를 위한 완전한 단일 페이지 애플리케이션(SPA)입니다. `getUserMedia`를 통해 카메라를 사용하고 WebGL을 이용해 실시간 시각 효과를 적용합니다.

### 주요 기능
- **카메라 접근:** 사용자의 카메라 비디오 스트림을 캡처합니다.
- **WebGL 렌더링:** WebGL을 사용하여 캔버스에 비디오 스트림을 그립니다.
- **효과:** GLSL 프래그먼트 셰이더를 사용하여 20가지 다양한 시각 효과(대왕 코, 홀쭉이, 뱅글뱅글, 만화경 등)를 구현합니다.
- **사진 촬영:** 현재 프레임을 PNG 이미지로 다운로드할 수 있는 스냅샷 기능을 제공합니다.
- **카운트다운:** 사진 촬영 전 시각적인 카운트다운을 표시합니다.
- **UI:** Tailwind CSS를 사용하여 반응형 디자인을 구현했습니다.

## 코드 구조
- **HTML:** 비디오 요소(숨김), 캔버스, 버튼, 효과 선택 목록을 포함한 레이아웃을 정의합니다.
- **CSS:** 스타일링을 위해 Tailwind CSS를 사용합니다. 커스텀 CSS는 애니메이션과 특정 효과 카드 스타일을 처리합니다.
- **JavaScript:**
  - `startCamera()`: 카메라 스트림을 초기화합니다.
  - `render()`: `requestAnimationFrame`을 통해 호출되는 메인 렌더링 루프입니다. WebGL 그리기를 처리합니다.
  - `initShaders()`: 정의된 모든 효과에 대한 셰이더 프로그램을 컴파일하고 링크합니다.
  - `createShader()`: 셰이더 컴파일을 돕는 헬퍼 함수입니다.
  - `buildList()`: 효과 선택을 위한 UI를 생성합니다.
  - 버튼에 대한 이벤트 리스너를 설정합니다.

## 식별된 문제점 및 개선 사항

### 1. 성능 (Performance)
- **반복적인 속성 조회:**
  `render` 함수는 매 프레임마다 활성 프로그램에 대해 `gl.getAttribLocation`과 `gl.getUniformLocation`을 호출합니다.
  ```javascript
  const pos = gl.getAttribLocation(p, "aPosition");
  const tc = gl.getAttribLocation(p, "aTexCoord");
  // ...
  gl.uniform1f(gl.getUniformLocation(p, "uTime"), time * 0.001);
  gl.uniform2f(gl.getUniformLocation(p, "uResolution"), canvas.width, canvas.height);
  ```
  **영향:** `getAttribLocation`과 `getUniformLocation`은 문자열 조회를 포함하는 상대적으로 비용이 많이 드는 작업입니다. 이를 매 프레임(초당 60회)마다 수행하는 것은 비효율적입니다.
  **제안:** 초기화 단계(`initShaders`)에서 이러한 위치를 캐싱하고 프로그램 객체와 함께 저장하세요.

### 2. 안정성 및 오류 처리 (Robustness & Error Handling)
- **셰이더 컴파일:**
  `createShader` 함수는 컴파일 상태를 확인하지만 실패 시 자세한 오류 로그 없이 `null`을 반환합니다.
  만약 `null`을 반환하면 `initShaders`의 `gl.attachShader`가 오류를 발생시키거나 조용히 실패하여 초기화가 중단될 수 있습니다.
- **프로그램 링크:**
  `gl.linkProgram`이 호출되지만 성공 여부를 확인하지 않습니다. 링크가 실패하면 프로그램이 유효하지 않게 되며 `render` 루프의 `gl.useProgram`이 실패합니다.
- **효과 폴백(Fallback) 부재:**
  특정 효과의 셰이더가 컴파일/링크에 실패하면 `programs` 객체에 유효하지 않은 항목이 생길 수 있습니다. 렌더 루프는 `programs[activeEffect]`를 사용하려고 시도하지만, 해당 프로그램이 유효하지 않거나 `null`인 경우 WebGL 오류가 발생하거나 앱이 멈출 수 있습니다.

### 3. 모범 사례 (Best Practices)
- **속성 활성화:**
  `gl.enableVertexAttribArray`가 매 프레임 호출됩니다. 위치 조회만큼 치명적이지는 않지만, 상태 관리를 효율적으로 하는 것이 좋습니다.
- **리소스 정리:**
  더 이상 필요하지 않은 셰이더나 프로그램을 삭제하는 메커니즘이 없습니다. 이 간단한 앱에서는 페이지 수명 동안 유지되므로 큰 문제는 아닙니다.

## 적용된 수정 사항
1.  **위치 캐싱:** `initShaders`를 수정하여 프로그램과 그 속성/유니폼 위치를 포함하는 객체를 반환하도록 변경했습니다.
2.  **오류 확인 추가:** `createShader`와 `initShaders`를 개선하여 컴파일이나 링크 실패 시 콘솔에 오류를 로그하도록 했습니다.
3.  **안전한 실패 처리:** 효과 로딩에 실패하면 해당 프로그램을 건너뛰거나 '원본(normal)' 효과로 대체하여 애플리케이션 충돌을 방지했습니다.

이러한 수정 사항은 코드에 이미 적용되었습니다.
