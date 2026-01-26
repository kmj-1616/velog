<h2 id="0-왜-nextjs와-funnel-패턴인가">0. 왜 Next.js와 Funnel 패턴인가?</h2>
<p>지난 포스팅에서 CSR과 SSR의 차이를 공부하며 Next.js에 대해 알아봤습니다. 저희 팀이 개발 중인 <strong>HandShake</strong> 서비스는 검색 엔진 최적화(SEO)와 빠른 초기 로딩 속도가 중요했기에 Next.js를 선택했고, 저는 그 중 서비스의 첫인상인 <strong>'온보딩(회원가입)'</strong> 구현을 맡게 되었습니다.</p>
<p>우리 서비스의 온보딩은 단순한 가입이 아니라, 직무 및 기술 스택 선택과 <strong>개발 성향 테스트(a.k.a. DSTI)</strong> 까지 총 3단계로 이어집니다. 이를 매끄럽게 구현하기 위해 <a href="https://www.youtube.com/watch?v=NwLWX2RNVcw">Toss 개발자 컨퍼런스 SLASH 23</a>을 보며 공부한 <strong>Funnel 패턴</strong>을 도입하여 상태 관리의 복잡도를 해결해 보고자 합니다.</p>
<hr />
<h2 id="1-흩어진-페이지의-한계">1. 흩어진 페이지의 한계</h2>
<p>Handshake에 소셜 로그인을 통해 처음 진입을 하면, 총 3단계의 프로필 정보 입력 과정을 거칩니다. 그래서 처음에는 단순히 /signup/1, /2, /3 이렇게 페이지를 나누면 되는 거 아닌가 생각했습니다.
하지만 이렇게 하면 몇 가지 불편한 점이 있습니다.</p>
<ul>
<li><strong>데이터 파편화</strong>: 1단계 기본 정보, 2단계 추가 정보, 3단계 개발 성향 유형 분석 결과를 마지막에 한꺼번에 API로 쏴야 하는데, 페이지가 갈리면 이 데이터를 들고 다니기가 매우 번거로워집니다. </li>
<li><strong>뒤로가기 제어</strong>: 사용자가 뒤로가기를 눌렀을 때의 데이터 보존 로직이 복잡해집니다.</li>
</ul>
<hr />
<h2 id="2-우리-서비스의-온보딩-및-api-흐름">2. 우리 서비스의 온보딩 및 API 흐름</h2>
<p>기획된 디자인 요구사항을 바탕으로 설계한 단계별 로직입니다.</p>
<h3 id="설문조사-단계-및-요구사항">설문조사 단계 및 요구사항</h3>
<ul>
<li><strong>1단계: 기본 정보</strong><ul>
<li>닉네임 중복 확인 (<code>POST /user/nickname</code>): 통과 필수</li>
<li>모달을 통한 직무 및 기술 스택 선택 (최대 5개)</li>
</ul>
</li>
<li>*<em>2단계: 네트워킹 정보 *</em><ul>
<li>네트워킹 목적, Github ID, 자기소개(선택)</li>
</ul>
</li>
<li><strong>3단계: 개발 성향 유형(DSTI) 분석 테스트(메인)</strong><ul>
<li>12개의 질문을 한 페이지당 하나씩 배치</li>
<li>답변 선택 시 자동으로 데이터 저장 후 다음 질문으로 이동</li>
</ul>
</li>
<li><strong>4단계: 분석 결과 및 완료</strong><ul>
<li>결과 카드(캐릭터 이미지 및 각 코드 설명) 확인 후 홈으로 이동 </li>
</ul>
</li>
</ul>
<h3 id="api-호출-시점">API 호출 시점</h3>
<p>모든 데이터를 들고 있다가 <strong>3단계가 끝나는 시점</strong>에 연속 호출합니다.</p>
<ol>
<li><code>POST /user/dsti</code>: 테스트 결과를 먼저 등록해서 DSTI 유형 생성</li>
<li><code>POST /user/info</code>: 수집된 모든 프로필 정보와 DSTI 결과(유형 코드+캐릭터 이미지)를 합쳐 최종 회원가입 완료</li>
</ol>
<hr />
<h2 id="3-프로젝트-폴더-구조">3. 프로젝트 폴더 구조</h2>
<p>프로젝트 규모가 커질 것을 대비해 <strong>Feature 중심 설계</strong>를 도입하여 <strong>도메인 단위의 응집도</strong>를 높였습니다. 동시에 백엔드와의 통신(Service) 및 데이터 규격(DTO)은 전역적으로 관리하여 데이터 모델의 <strong>일관성</strong>을 유지할 수 있도록 구성했습니다.</p>
<pre><code>src/
├── app/
│   └── (auth)/               # 라우트 그룹 (URL에 포함되지 않음)
│       └── login/page.tsx    # 소셜 로그인 페이지
│       └── signup/
│           └── page.tsx      # 온보딩 컨트롤 타워 (useFunnel 활용)
├── components/
│   └── ui/                   # 프로젝트 공통 UI (Button, Input, Progress 등)
├── features/
│   └── auth/                 # 🔵 인증/회원가입 관련 도메인 응집 (UI/로직)
│       ├── components/       # 각 가입 단계별 UI 
│       │   └── Step1Profile.tsx/ 
│       │   └── Step2Networking.tsx/ 
│       │   └── Step3DSTI.tsx/ 
│       └── hooks/            # Funnel 제어 등 도메인 특화 로직
│           └── useFunnel.tsx/ 
├── services/                 # 🟠 백엔드 API 통신 
│   ├── api.ts                
│   └── auth/               
│   │   └── api.ts/           # 인증 관련 API 엔드포인트 
│   │   └── hooks.ts/         # React-Query와 연결된 API 호출 훅
│   └── user/               
│       └── api.ts/           # 유저 정보 관련 API 엔드포인트 함수 정의
│       └── hooks.ts/         # React-Query와 연결된 API 호출 훅
├── types/                    # ⭐ API 응답 및 데이터 공통 타입 (DTO)
│   ├── auth.ts               # 인증 관련 타입 
│   └── user.ts               # 유저 프로필, 경력, 기술스택 등 핵심 데이터 타입
├── assets/icons              # 아이콘, 이미지 등 정적 파일
└── utils/                    # 토큰 관련 유틸</code></pre><h4 id="💡-포인트">💡 포인트</h4>
<ul>
<li><strong>Feature 기반 UI 응집</strong>: features/auth 내부에는 오직 해당 도메인의 <strong>화면 구성(Components)</strong>과 <strong>상태 흐름(Hooks)</strong>만 두어 UI 복잡도를 낮췄습니다.</li>
<li><strong>서비스 계층 분리</strong>: 여러 페이지나 다른 기능에서도 인증 관련 API를 재사용하기 쉽게 설계했습니다.</li>
<li><strong>DTO 중앙 관리</strong>: 백엔드 명세가 변경될 경우 src/types 폴더 내의 파일만 수정하면 프로젝트 전체에 반영되도록 관리 포인트를 일원화했습니다.</li>
</ul>
<hr />
<h2 id="4-핵심-구현-usefunnel-커스텀-훅">4. 핵심 구현: <code>useFunnel</code> 커스텀 훅</h2>
<p>Toss의 <code>useFunnel</code> 라이브러리 콘셉트를 참고해서, <strong>현재 단계(step)를 관리</strong>하고 해당 단계일 때만 화면을 보여주는 컴포넌트를 반환하도록 설계했습니다.</p>
<pre><code>import { useState } from 'react';

export type SignupStep = 'step1' | 'step2' | 'step3' | 'step4';

export function useFunnel(defaultStep: SignupStep) {
  const [step, setStep] = useState&lt;SignupStep&gt;(defaultStep);

  const Step = ({ name, children }: { name: SignupStep; children: React.ReactNode }) =&gt; {
    return step === name ? &lt;&gt;{children}&lt;/&gt; : null;
  };

  return { step, setStep, Step };
}</code></pre><h4 id="💡-포인트-1">💡 포인트</h4>
<p>Toss의 <code>useFunnel</code>은 내부적으로 <code>React.Children</code>을 순회하며 현재 단계에 맞는 자식을 찾아내는 고도화된 인터페이스를 제공합니다. 하지만 라이브러리의 복잡한 내부 로직을 그대로 가져오기보다, <code>Step</code> 컴포넌트를 직접 정의하여 반환함으로써 <strong>구현은 단순화하되, Toss가 지향하는 '선언적인 코드'와 '응집도'</strong> 는 그대로 가져올 수 있도록 커스터마이징했습니다.
단순히 <code>step === 'step1' &amp;&amp; &lt;Component /&gt;</code>라고 쓸 수도 있지만, 이렇게 컴포넌트로 추상화하면 부모 페이지의 JSX가 마치 <strong>'하나의 큰 폼을 단계별로 명세한 것'</strong> 처럼 읽혀 가독성이 좋아집니다.</p>
<hr />
<h2 id="5-메인-컨트롤-타워-signuppagetsx">5. 메인 컨트롤 타워: <code>signup/page.tsx</code></h2>
<p>여기가 <strong>응집도</strong>의 핵심으로, 모든 단계의 데이터가 이 페이지의 <code>formData</code> 하나에 모입니다.</p>
<pre><code>export default function SignupPage() {
  const { Step, setStep, step } = useFunnel('step1');
  const [formData, setFormData] = useState&lt;UserProfile&gt;(INITIAL_DATA);

  // 데이터 업데이트 핸들러
  const updateFormData = (newData: Partial&lt;UserProfile&gt;) =&gt; {
    setFormData((prev) =&gt; ({ ...prev, ...newData }));
  };

  return (
    &lt;main&gt;
      &lt;Progress value={calculateProgress(step)} /&gt;

      &lt;Step name=&quot;step1&quot;&gt;
        &lt;Step1Profile 
          data={formData} 
          onNext={(data) =&gt; { updateFormData(data); setStep('step2'); }} 
        /&gt;
      &lt;/Step&gt;

      &lt;Step name=&quot;step2&quot;&gt;
        &lt;Step2Networking 
          data={formData} 
          onNext={(data) =&gt; { updateFormData(data); setStep('step3'); }} 
          onPrev={() =&gt; setStep('step1')} 
        /&gt;
      &lt;/Step&gt;

      {/* ... 반복 ... */}
    &lt;/main&gt;
  );
}</code></pre><hr />
<h2 id="6-학습-포인트-및-회고">6. 학습 포인트 및 회고</h2>
<blockquote>
</blockquote>
<ul>
<li><strong>데이터 흐름의 단방향성 확보</strong>: 각 단계별 컴포넌트가 비즈니스 로직을 직접 수행하지 않고, 입력받은 데이터를 부모에게 넘겨주는 구조를 택했습니다. 덕분에 데이터 흐름이 단방향으로 명확해졌고, 복잡한 온보딩 과정에서도 상태 추적이 용이해졌습니다.</li>
<li><strong>선언적인 UI와 추상화의 힘</strong>: <code>{step === 'step1' &amp;&amp; &lt;Step1 /&gt;}</code> 같은 조건부 렌더링 대신, <code>&lt;Step name=&quot;step1&quot;&gt;</code> 컴포넌트로 감싸주었습니다. 이를 통해 JSX 가독성이 상승했고, 코드가 마치 <strong>'단계별 명세서'</strong>처럼 읽히는 추상화의 이점을 확인할 수 있습니다.</li>
<li><strong>사용자 경험(UX)</strong>: Next.js의 App Router 환경에서 실제 페이지를 이동하지 않고 컴포넌트만 교체되므로, 웹이지만 앱처럼 끊김 없는 전환 효과를 줄 수 있었습니다.</li>
<li><strong>가입 절차의 원자성</strong>: 데이터 누락이나 중간 이탈로 인한 불완전한 회원 정보 생성을 방지하기 위해, 중간 저장 방식 대신 마지막 단계에서 모든 데이터를 한 번에 처리하는 전략을 세웠습니다. 이는 DB의 정합성을 지키는 데에도 유리합니다. </li>
</ul>
<p>단순히 기능을 구현하는 것을 넘어, <strong>&quot;어떻게 하면 더 읽기 쉬운 코드를 짤 것인가?&quot;</strong>와 <strong>&quot;어떻게 하면 중복을 줄이고 재사용할 수 있을까?&quot;</strong>를 고민해볼 수 있는 시간이었습니다.
Toss의 좋은 레퍼런스를 우리 프로젝트의 규모에 맞게 적절히 변형하여 적용해본 의미 있는 경험이었습니다.</p>