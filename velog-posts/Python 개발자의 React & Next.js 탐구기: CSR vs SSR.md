<p>싸피에서의 지난 6개월간 웹 개발을 할 때 Python, Vue.js, Django를 사용했다. 그런데 이번 프로젝트에서는 Java와 React 기반으로 진행하게 되었다. 프론트엔드 팀원들이 React와 Next.js 중 어떤 걸로 할지 의견을 나누는데, 그 사이에서 난 &quot;두 가지가 차이가 뭐길래?&quot;라는 궁금증이 생겼다.</p>
<p>이번 프로젝트에서는 백엔드를 주로 담당하지만, 전체적인 기술 구조도 이해하면서 프론트엔드도 참여해 보고 싶었기에 이 부분은 풀고 가야 할 문제다. 이 문제의 해결을 위해 CSR과 SSR을 공부해오라는 숙제(?)를 받았고, 공부하면서 이해한 내용을 정리해보려고 한다.</p>
<hr />
<h2 id="1-근본적인-차이-라이브러리-vs-프레임워크">1. 근본적인 차이: 라이브러리 vs 프레임워크</h2>
<p>React와 Next.js를 비교할 때 가장 먼저 알아야 할 것은 두 도구의 '성격'이다. </p>
<h3 id="react-library">React (Library)</h3>
<p>React는 <strong>'UI를 만들기 위한 도구'</strong> 이다.
레고 블록 같은 컴포넌트 기반 개발이기 때문에, 재사용성과 복잡한 코드의 단순화와 같은 장점이 있다.
하지만 라우팅(React Router), 상태 관리(Redux/Recoil), 빌드 설정(Webpack/Vite) 등을 개발자가 직접 선택하고 조합해야 한다는 한계점이 있다. 자유도가 높지만 설정 책임도 개발자에게 있다는 것이다. </p>
<h3 id="nextjs-framework">Next.js (Framework)</h3>
<p>이에 비해 Next.js는 React를 기반으로 한 <strong>'풀스택 프레임워크'</strong> 이다.
React의 한계를 보완하면서, 정해진 폴더 구조를 따르면 라우팅이 자동으로 되고, SSR(서버사이드 렌더링)/SSG(정적 사이트 생성)/ISR(증분 정적 재생성) 등 다양한 렌더링 전략을 기본 제공한다. 
그래서 특히 SEO(검색 엔진 최적화)와 초기 로딩 속도 최적화에 강점을 가지고 있다. </p>
<hr />
<h2 id="2-csr-vs-ssr-렌더링-방식의-차이">2. CSR vs SSR (렌더링 방식의 차이)</h2>
<p>그렇다면 두 도구를 선택하는 가장 큰 기준은 무엇일까? 바로 <strong>'화면을 어디서 그리는가?'</strong> 에 있다. </p>
<h3 id="클라이언트-사이드-렌더링-csr">클라이언트 사이드 렌더링 (CSR)</h3>
<ul>
<li><strong>주체</strong>: 브라우저(Client)</li>
<li><strong>과정</strong>: 서버는 빈 <code>index.html</code>과 JS 파일만 보낸다. 브라우저가 JS를 실행해 DOM을 그린다.</li>
<li><strong>장점</strong>: 첫 로딩 후 페이지 이동이 매우 빠르고 서버 부담이 적다.</li>
<li><strong>단점</strong>: JS 파일이 커지면 첫 화면을 보기까지 오래 걸리고, 검색 엔진이 빈 페이지로 오해할 수 있다. </li>
</ul>
<h3 id="서버-사이드-렌더링-ssr">서버 사이드 렌더링 (SSR)</h3>
<ul>
<li><strong>주체</strong>: 서버(Node.js Server)</li>
<li><strong>과정</strong>: 사용자가 요청하면 <strong>서버에서 데이터를 채운 HTML</strong>을 미리 만들어 보낸다. 서버에서 완성된 HTML을 먼저 보여주고, 그 위에 자바스크립트를 입혀 대화형 기능을 활성화하는 Hydration 과정을 거치게 된다. </li>
<li><strong>장점</strong>: 초기 로딩 속도가 빨라 사용자 경험이 좋고, 검색 엔진이 읽기 최적화되어 있다(SEO). (그래서 요즘 대부분의 서비스가 Node.js를 사용한다고 한다.)</li>
<li><strong>단점</strong>: 페이지 이동 시마다 서버 요청이 발생할 수 있고, 서버 리소스 비용이 발생한다.</li>
</ul>
<hr />
<h2 id="3-언제-무엇을-써야-할까">3. 언제 무엇을 써야 할까?</h2>
<table>
<thead>
<tr>
<th align="left">구분</th>
<th align="left">React (Pure CSR)</th>
<th align="left">Next.js (SSR/Hybrid)</th>
</tr>
</thead>
<tbody><tr>
<td align="left"><strong>렌더링 방식</strong></td>
<td align="left">Client Side</td>
<td align="left">Server Side / Static / Client (혼합)</td>
</tr>
<tr>
<td align="left"><strong>SEO</strong></td>
<td align="left">불리함 (별도 작업 필요)</td>
<td align="left"><strong>매우 유리함</strong></td>
</tr>
<tr>
<td align="left"><strong>라우팅</strong></td>
<td align="left">React Router 설치 필요</td>
<td align="left">파일 시스템 기반 (자동)</td>
</tr>
<tr>
<td align="left"><strong>초기 로딩</strong></td>
<td align="left">상대적으로 느림 (JS 로드 대기)</td>
<td align="left"><strong>매우 빠름</strong> (HTML 즉시 렌더링)</td>
</tr>
<tr>
<td align="left"><strong>주요 활용</strong></td>
<td align="left">대시보드, 개인화 앱, 관리자 페이지</td>
<td align="left">이커머스, 블로그, 마케팅 페이지</td>
</tr>
</tbody></table>
<hr />
<h2 id="백엔드-관점에서의-결정적-차이-서버가-하나-더-생겼다">백엔드 관점에서의 결정적 차이: &quot;서버가 하나 더 생겼다&quot;</h2>
<p>내가 느낀 두 방식의 결정적 차이는 <strong>인프라 아키텍처의 변화</strong>이다.</p>
<ol>
<li><p><strong>React(CSR) 환경:</strong></p>
<ul>
<li>백엔드는 단순 <strong>Rest API 서버</strong> 역할만 수행한다. </li>
<li>CORS 설정만 잘 해주면 프론트엔드와 깔끔하게 분리된다.</li>
</ul>
</li>
<li><p><strong>Next.js(SSR) 환경:</strong></p>
<ul>
<li>프론트엔드 영역에도 <strong>Node.js 서버</strong>가 존재하게 된다.</li>
<li><strong>인증(Auth):</strong> 쿠키/세션 처리 시 '브라우저-Next서버-Java서버' 간의 토큰 전달 과정을 고민해야 한다.</li>
<li><strong>통신:</strong> 브라우저에서 Java로 직접 호출할지, Next 서버를 프록시(Proxy)로 거칠지 설계가 필요하다.</li>
</ul>
</li>
</ol>
<hr />
<h2 id="마치며">마치며</h2>
<p>결국 &quot;React냐 Next.js냐&quot;의 선택은 우리가 만드는 서비스가 <strong>'사용자 인터랙션이 중심인 서비스인가?(React)' 또는 '초기 로딩 속도가 중요한가?(Next.js)</strong> 에 달려 있다. </p>
<p>백엔드 개발자로서도 프론트엔드가 데이터를 가져가는 방식을 이해하니 전체적인 시스템 아키텍처를 설계하는 데 큰 도움이 된 공부였다. 이제 이 개념을 바탕으로 본격적인 프로젝트 세팅에 들어가 보자! </p>