<h2 id="테스트-데이터의-설계">테스트 데이터의 설계</h2>
<p>저희 프로젝트의 AI 담당 팀원이 추천 시스템의 로직을 크게 개선했습니다. 메타 정보와 텍스트를 결합한 <strong>하이브리드 임베딩</strong> 구조를 적용하고, <strong>Word2Vec</strong> 모델을 재학습하여 추천 품질을 높이는 작업이었습니다.</p>
<p>하지만 모델이 아무리 좋아도, 실제 서비스에서 마주할 다양한 유저 케이스를 검증하지 못하면 성능이 좋다고 할 수 없습니다. 특히 신규 유저에게 적절한 추천을 제공하는 <strong>Cold Start</strong>는 항상 어려운 문제인 것 같습니다.</p>
<p>저는 저희 서비스에서 가장 중요한 AI 추천 모델의 성능을 최대한으로 테스트하고, 학습 품질을 안정화하기 위해 <strong>240명의 가상 페르소나를 설계하고 데이터 파이프라인을 구축</strong>했습니다.</p>
<hr />
<h2 id="4가지-유저-페르소나">4가지 유저 페르소나</h2>
<p>단순히 랜덤한 값을 생성하는 것이 아니라, 모델의 강건성을 검증하기 위해 데이터의 특성 조합을 4가지 타입으로 세분화했습니다.</p>
<table>
<thead>
<tr>
<th align="left">유저 타입</th>
<th align="left">인원</th>
<th align="left">특징</th>
<th align="left">설계 의도</th>
<th align="left">목적</th>
</tr>
</thead>
<tbody><tr>
<td align="left"><strong>Cold Start</strong></td>
<td align="left">50</td>
<td align="left">데이터가 적은 막 가입한 유저</td>
<td align="left">데이터가 희소한 신규 유저 대응</td>
<td align="left">초기 탐색용 프로필의 추천 수렴 속도 분석</td>
</tr>
<tr>
<td align="left"><strong>Core Cluster</strong></td>
<td align="left">100</td>
<td align="left">직무 1개에 깊은 스택을 보유한 전형적인 개발자</td>
<td align="left">추천 시스템의 기준점인 전형적인 전문가 그룹 인식</td>
<td align="left">하이브리드 임베딩의 벡터 군집화 성능 검증</td>
</tr>
<tr>
<td align="left"><strong>Bridge</strong></td>
<td align="left">60</td>
<td align="left">여러 직무와 스택을 가진 하이브리드형</td>
<td align="left">직무 간 연관성 학습 (ex. FE+기획 등)</td>
<td align="left">다중 레이블 환경에서의 추천 확장성 테스트</td>
</tr>
<tr>
<td align="left"><strong>Outlier</strong></td>
<td align="left">30</td>
<td align="left">직무는 A인데, 스택은 B인 특이 케이스</td>
<td align="left">직무와 스택이 불일치하는 노이즈</td>
<td align="left">모델의 예외 처리 및 강건성 검증</td>
</tr>
</tbody></table>
<hr />
<h2 id="비즈니스-제약-조건을-코드로-구현하기">비즈니스 제약 조건을 코드로 구현하기</h2>
<p>데이터 엔지니어링 단계에서 가장 고민했던 점은 <strong>서비스의 제약 조건을 준수하면서도 유의미한 변동성을 만드는 것</strong>이었습니다.</p>
<h4 id="①-context-aware-스택-매핑">① Context-Aware 스택 매핑</h4>
<p>단순 랜덤 생성을 배제하고, <code>position_stacks</code> 매핑 테이블을 구축했습니다. 이는 AI 모델이 &quot;백엔드 개발자는 Java/Spring 스택을 가질 확률이 높다&quot;는 실제 문맥을 학습할 수 있습니다.</p>
<h4 id="②-예외-처리를-통한-데이터-무결성-확보">② 예외 처리를 통한 데이터 무결성 확보</h4>
<p>데이터 생성 중 발생할 수 있는 논리적 오류를 방지하기 위해 <code>min()</code> 함수를 활용했습니다. 예를 들어 설계 상 매핑된 스택이 3개뿐인 직무(ex. 기획/디자인)에서 5개를 샘플링하려 할 때 발생할 수 있는 <code>ValueError</code>를 방지해 파이프라인의 안정성을 높였습니다.</p>
<pre><code class="language-python"># 기술 스택이 부족하더라도 에러 없이 가능한 범위 내에서 최대치를 추출하도록 설계
skills = random.sample(available_skills, min(len(available_skills), random.randint(3, 5)))</code></pre>
<h4 id="③-bridge-유저-설계">③ Bridge 유저 설계</h4>
<p>저희 서비스의 기획상 직무는 최대 3개까지 선택이 가능합니다. 이를 반영해 2~3개의 직무를 가진 <strong>Bridge 유저</strong>를 설계했습니다. 이 타입은 하이브리드 임베딩이 '프론트엔드 개발자이면서 디자인 툴도 다루는 유저' 같은 경우의 벡터를 얼마나 정교하게 생성하는지를 테스트할 수 있습니다.</p>
<hr />
<h2 id="데이터-생성-파이썬-코드">데이터 생성 파이썬 코드</h2>
<p>아래는 실제 시뮬레이션 환경 구축에 사용한 전체 코드입니다. <code>pandas</code>를 활용해 최종적으로 엑셀 파일로 추출해 DB 적재를 준비했습니다.</p>
<pre><code class="language-python">import pandas as pd
import random

# 1. 기초 데이터 및 상수 설정
dsti_dims = [['P', 'E'], ['B', 'D'], ['W', 'A'], ['U', 'R']]
careers = ['employed', 'job_seeking', 'freelancer', 'student']
networks_pool = ['coffee_chat', 'study_group', 'side_project']

position_stacks = {
    &quot;frontend_developer&quot;: [&quot;javascript&quot;, &quot;typescript&quot;, &quot;react&quot;, &quot;vue&quot;, &quot;nextjs&quot;, &quot;svelte&quot;],
    &quot;backend_developer&quot;: [&quot;java&quot;, &quot;spring_framework&quot;, &quot;python&quot;, &quot;django&quot;, &quot;fastapi&quot;, &quot;nodejs&quot;, &quot;nestjs&quot;, &quot;go_language&quot;],
    &quot;data_ai_engineer&quot;: [&quot;mysql&quot;, &quot;postgresql&quot;, &quot;mongodb&quot;, &quot;redis_cache&quot;, &quot;oracle_database&quot;],
    &quot;game_developer&quot;: [&quot;pytorch&quot;, &quot;tensorflow&quot;, &quot;scikit_learn&quot;, &quot;pandas&quot;, &quot;spark_engine&quot;, &quot;hadoop_platform&quot;],
    &quot;embedded_systems_engineer&quot;: [&quot;csharp_language&quot;, &quot;cpp_language&quot;, &quot;unity_engine&quot;, &quot;unreal_engine&quot;, &quot;opengl_api&quot;, &quot;c_language&quot;, &quot;raspberry_pi_platform&quot;, &quot;arduino_platform&quot;],
    &quot;mobile_app_developer&quot;: [&quot;swift_language&quot;, &quot;kotlin&quot;, &quot;flutter&quot;, &quot;react_native&quot;],
    &quot;devops_infra_engineer&quot;: [&quot;aws&quot;, &quot;docker&quot;, &quot;kubernetes&quot;, &quot;firebase&quot;, &quot;jenkins&quot;, &quot;github_actions&quot;],
    &quot;product_planning_design&quot;: [&quot;figma_design_tool&quot;, &quot;photoshop_design_tool&quot;, &quot;illustrator_design_tool&quot;]
}

positions_list = list(position_stacks.keys())

def get_dsti():
    return &quot;&quot;.join([random.choice(d) for d in dsti_dims])

def get_common_meta(is_cold=False):
    career = random.choice(careers)
    exp = str(random.choice([True, False])).lower()
    net_count = 1 if is_cold else random.choices([1, 2, 3], weights=[0.5, 0.3, 0.2])[0]
    networks = &quot;, &quot;.join(random.sample(networks_pool, net_count))
    return career, exp, networks

# 2. 유저 페르소나 유형별 데이터 생성 로직

data = []
user_idx = 1

for _ in range(50):
    pos = random.choice(positions_list)

    # 스킬은 해당 직무에서 1~2개만 선택
    available_skills = position_stacks[pos]
    skills = random.sample(available_skills, min(len(available_skills), random.randint(1, 2)))

    career, exp, networks = get_common_meta(is_cold=True)

    data.append({
        &quot;type&quot;: &quot;Cold Start&quot;,
        &quot;nickname&quot;: f&quot;dummy_{user_idx:03d}&quot;,
        &quot;positions&quot;: pos,
        &quot;tech_skills&quot;: &quot;, &quot;.join(skills),
        &quot;career&quot;: career,
        &quot;experience&quot;: exp,
        &quot;networks&quot;: networks,
        &quot;dsti&quot;: get_dsti()
    })
    user_idx += 1

for _ in range(100):
    # 포지션을 순차적으로 돌면서 골고루 생성
    pos = positions_list[user_idx % len(positions_list)]

    # 해당 직무의 스택을 깊게(3~5개) 선택
    available_skills = position_stacks[pos]
    skills = random.sample(available_skills, min(len(available_skills), random.randint(3, 5)))

    career, exp, networks = get_common_meta()

    data.append({
        &quot;type&quot;: &quot;Core Cluster&quot;,
        &quot;nickname&quot;: f&quot;dummy_{user_idx:03d}&quot;,
        &quot;positions&quot;: pos,
        &quot;tech_skills&quot;: &quot;, &quot;.join(skills),
        &quot;career&quot;: career,
        &quot;experience&quot;: exp,
        &quot;networks&quot;: networks,
        &quot;dsti&quot;: get_dsti()
    })
    user_idx += 1

for _ in range(60):
    # 직무 2~3개 랜덤 선택
    pos_count = random.choices([2, 3], weights=[0.7, 0.3])[0]
    selected_positions = random.sample(positions_list, pos_count)

    # 선택된 직무들의 스택 풀을 모두 합침
    combined_skill_pool = []
    for p in selected_positions:
        combined_skill_pool.extend(position_stacks[p])

    # 합친 풀에서 중복 제거 후 4~5개 스택 랜덤 선택
    combined_skill_pool = list(set(combined_skill_pool))
    pick_count = min(len(combined_skill_pool), random.randint(4, 5))
    final_skills = random.sample(combined_skill_pool, pick_count)

    career, exp, networks = get_common_meta()

    data.append({
        &quot;type&quot;: &quot;Bridge&quot;,
        &quot;nickname&quot;: f&quot;dummy_{user_idx:03d}&quot;,
        &quot;positions&quot;: &quot;, &quot;.join(selected_positions),
        &quot;tech_skills&quot;: &quot;, &quot;.join(final_skills),
        &quot;career&quot;: career,
        &quot;experience&quot;: exp,
        &quot;networks&quot;: networks,
        &quot;dsti&quot;: get_dsti()
    })
    user_idx += 1

for _ in range(30):
    # 메인 직무 1개 선택
    main_pos = random.choice(positions_list)

    # 다른 직무들의 스택 풀 생성
    other_positions = [p for p in positions_list if p != main_pos]
    other_pos = random.choice(other_positions)

    # 내 직무 스택 1개 + 다른 직무 스택 2~3개
    my_skill = random.sample(position_stacks[main_pos], 1)
    other_pool = position_stacks[other_pos]
    weird_skills = random.sample(other_pool, min(len(other_pool), random.randint(2, 3)))

    final_skills = my_skill + weird_skills
    random.shuffle(final_skills)

    career, exp, networks = get_common_meta()

    data.append({
        &quot;type&quot;: &quot;Outlier&quot;,
        &quot;nickname&quot;: f&quot;dummy_{user_idx:03d}&quot;,
        &quot;positions&quot;: main_pos,
        &quot;tech_skills&quot;: &quot;, &quot;.join(final_skills),
        &quot;career&quot;: career,
        &quot;experience&quot;: exp,
        &quot;networks&quot;: networks,
        &quot;dsti&quot;: get_dsti()
    })
    user_idx += 1

# 3. 파일 저장
df = pd.DataFrame(data)
df.to_excel(&quot;dummy_users.xlsx&quot;, index=False)</code></pre>
<hr />
<h2 id="결과-및-회고">결과 및 회고</h2>
<p>팀원의 모델 개선 작업(Word2Vec 재학습 및 액션 로그 반영) 과정에서 제가 생성한 더미 데이터는 다음과 같은 역할을 했습니다.</p>
<ul>
<li><strong>가중치 최적화</strong>: 메타 정보와 텍스트 임베딩의 결합 가중치를 조정할 때, 의도적으로 설계된 Bridge 유저들의 추천 결과를 보며 로직을 미세 조정(Fine-tuning)할 수 있었습니다.</li>
</ul>
<ul>
<li><strong>데이터 표준화</strong>: 엑셀 데이터를 DB에 적재하는 과정에서 <code>meta_key</code> 소문자 통일, <code>experience</code> 값의 boolean 타입 처리 등 데이터 엔지니어링 측면의 정제를 수행하여 추천 API 인풋 데이터의 안정성을 확보했습니다.</li>
</ul>
<p>이번 작업을 통해 <strong>모델링만큼 중요한 것이 데이터의 분포 설계</strong>라는 것을 배웠습니다. 데이터 분석가는 단순히 주어진 데이터를 해석하기보다, 모델이 제대로 성능을 발휘하기 위해 데이터를 잘 설계할 수 있어야 함을 느낄 수 있었던 경험이었습니다. </p>