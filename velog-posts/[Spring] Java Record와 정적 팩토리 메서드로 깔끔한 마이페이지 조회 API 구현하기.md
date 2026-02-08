<p>저희 <strong>Handshake</strong> 서비스의 마이페이지는 유저의 기본적인 프로필 정보뿐만 아니라, 포지션, 기술 스택, 네트워킹 목적 등 여러 연관 테이블의 데이터를 모아서 보여줘야 합니다.
이번 포스팅에서는 마이페이지 조회 API를 구현하고, 서비스 레이어의 가독성을 높이기 위해 <strong>정적 팩토리 메서드(Static Factory Method)</strong> 로 리팩토링한 과정을 공유하려고 합니다.</p>
<hr />
<h2 id="초기-api-구현">초기 API 구현</h2>
<p>가장 먼저 <code>Controller</code>, <code>Service</code>, <code>Repository</code>, <code>Response DTO</code>를 각각 작성하여 기능을 구현했습니다.</p>
<h3 id="📁-profileresponse-dto">📁 ProfileResponse (DTO)</h3>
<p>사용자 정보와 각 중간 테이블에서 가져올 ID 리스트를 담기 위해 Java의 Record 구조를 사용했습니다. 초기에는 <code>@Builder</code>를 사용하여 객체를 생성하도록 설계했습니다.</p>
<pre><code class="language-java">@Builder
public record ProfileResponse(
        String nickname,
        String profileImageUrl,
        Boolean experience,
        String career,
        String dsti,
        List&lt;Long&gt; positions,
        List&lt;Long&gt; techSkills,
        List&lt;Long&gt; networks,
        String githubId,
        String selfIntro
) { }</code></pre>
<h3 id="📁-userrepository--userprofileservice">📁 UserRepository &amp; UserProfileService</h3>
<p>먼저 <code>UserNetworkRepository</code>, <code>UserPositionRespository</code>, <code>UserTechSkillRepository</code> 에서 사용자 본체의 정보를 가져오는 쿼리 메서드를 추가했습니다.
<code>USerProfileService</code> 에서는 연관된 모든 ID 리스트를 취합하는 로직을 작성했습니다.</p>
<pre><code class="language-java">@Transactional(readOnly = true)
public ProfileResponse getMyProfile(Long userId) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -&gt; new GlobalException(ErrorCode.USER_NOT_FOUND));

    // 각 중간 테이블에서 ID 리스트 추출
    List&lt;Long&gt; positionIds = userPositionRepository.findAllByUserId(userId).stream()
            .map(up -&gt; up.getPosition().getId()).toList();
    List&lt;Long&gt; techSkillIds = userTechSkillRepository.findAllByUserId(userId).stream()
            .map(up -&gt; up.getTechSkill().getId()).toList();
    List&lt;Long&gt; networkIds = userNetworkRepository.findAllByUserId(userId).stream()
            .map(up -&gt; up.getNetwork().getId()).toList();

    // 빌더 패턴을 사용하여 DTO 생성 (리팩토링 전)
    return ProfileResponse.builder()
            .nickname(user.getNickname())
            .profileImageUrl(user.getProfileImageUrl())
            .experience(user.getExperience())
            .career(user.getCareer())
            .dsti(user.getDsti())
            .positions(positionIds)
            .techSkills(techSkillIds)
            .networks(networkIds)
            .githubId(user.getGithubId())
            .selfIntro(user.getSelfIntro())
            .build();
}</code></pre>
<h3 id="📁-usercontroller">📁 UserController</h3>
<p><code>GET /user/info</code> 엔드포인트를 생성해 현재 인증된 유저의 정보를 반환하도록 연결했습니다.</p>
<pre><code class="language-java">@GetMapping(&quot;/info&quot;)
public ResponseBody&lt;ProfileResponse&gt; getInfo(@AuthenticationPrincipal Long userId) {
    ProfileResponse response = userProfileService.getMyProfile(userId);
    return ResponseBody.success(response);
}</code></pre>
<hr />
<h2 id="기존-구조의-문제점">기존 구조의 문제점</h2>
<p>위와 같이 구현했을 때 기능상 문제는 없었지만, 팀원의 코드 리뷰를 통해 몇 가지 아쉬운 점을 발견했습니다.</p>
<ol>
<li><p><strong>서비스 로직의 비대화</strong>: 서비스 레이어에서 DTO의 필드를 하나하나 매핑하다 보니 코드가 길어지고 가독성이 떨어졌습니다.</p>
</li>
<li><p><strong>캡슐화 부족</strong>: 응답 형식을 생성하는 책임이 서비스 레이어에 노출되어 있어, 필드가 추가되거나 변경될 때 서비스 코드를 매번 수정해야 했습니다.</p>
</li>
<li><p><strong>디버깅의 어려움</strong>: 빌더가 길어질수록 어떤 데이터가 어디서 잘못 매핑되었는지 한눈에 파악하기 어려웠습니다.</p>
</li>
</ol>
<p>이를 해결하기 위해 <strong>정적 팩토리 메서드(Static Factory Method)</strong> 구조로 리팩토링을 진행했습니다.</p>
<hr />
<h2 id="리팩토링-정적-팩토리-메서드-도입">리팩토링: 정적 팩토리 메서드 도입</h2>
<h3 id="🛠️-profileresponse-수정">🛠️ ProfileResponse 수정</h3>
<p>DTO 내부에 유저 엔티티와 ID 리스트들을 받아 직접 객체를 생성하는 <code>from</code> 메서드를 추가했습니다.</p>
<pre><code class="language-java">@Builder
public record ProfileResponse(
    // 필드 생략
) {
    public static ProfileResponse from(User user,
                                       List&lt;Long&gt; positionIds,
                                       List&lt;Long&gt; techSkillIds,
                                       List&lt;Long&gt; networkIds) {
        return ProfileResponse.builder()
                .nickname(user.getNickname())
                .profileImageUrl(user.getProfileImageUrl())
                .experience(user.getExperience())
                .career(user.getCareer())
                .dsti(user.getDsti())
                .positions(positionIds)
                .techSkills(techSkillIds)
                .networks(networkIds)
                .githubId(user.getGithubId())
                .selfIntro(user.getSelfIntro())
                .build();
    }
}</code></pre>
<h3 id="🛠️-userprofileservice-수정">🛠️ UserProfileService 수정</h3>
<p>이제 서비스단에서는 복잡한 빌더 로직이 사라지고, 단 한 줄로 조회가 가능해졌습니다.</p>
<pre><code class="language-java">@Transactional(readOnly = true)
public ProfileResponse getMyProfile(Long userId) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -&gt; new GlobalException(ErrorCode.USER_NOT_FOUND));

    List&lt;Long&gt; positionIds = getPositionIds(userId);
    List&lt;Long&gt; techSkillIds = getTechSkillIds(userId);
    List&lt;Long&gt; networkIds = getNetworkIds(userId);

    // 정적 팩토리 메서드로 깔끔하게 반환
    return ProfileResponse.from(user, positionIds, techSkillIds, networkIds);
}</code></pre>
<hr />
<h2 id="정적-팩토리-메서드-도입의-장점">정적 팩토리 메서드 도입의 장점</h2>
<p>이렇게 정적 팩토리 메서드로 리팩토링을 해서 얻은 장점은 여러가지가 있습니다.</p>
<blockquote>
</blockquote>
<ol>
<li><strong>명확한 의도 표현</strong>: <code>from()</code>이라는 이름만 봐도 &quot;User 객체로부터 Response DTO를 만든다&quot;는 의도가 직관적으로 전달됩니다.</li>
<li><strong>변환 책임의 캡슐화</strong>: 엔티티를 DTO로 변환하는 상세 로직이 DTO 내부에 숨겨져 있어 서비스 코드가 간결해집니다.</li>
<li><strong>Builder를 통한 가독성과 유연성</strong>: 메서드 내부에서는 Builder를 사용하므로 필드가 많아도 가독성이 유지되며, 파라미터 순서로 인한 실수를 방지할 수 있습니다.</li>
<li><strong>유지보수의 용이성</strong>: 필드가 추가되거나 변경되어도 DTO 내부의 <code>from</code> 메서드만 수정하면 됩니다. 서비스 레이어까지 영향이 가지 않습니다.</li>
<li><strong>명시적인 설계 활용</strong>: <code>of</code>, <code>from</code> 등 관례적인 이름을 사용하여 다른 개발자가 코드를 볼 때도 의도를 쉽게 파악할 수 있습니다.<blockquote>
</blockquote>
</li>
</ol>
<p>이번 리팩토링을 통해 단순히 기능을 구현하는 것을 넘어, <strong>&quot;어떻게 하면 더 읽기 좋고 관리하기 쉬운 코드를 만들 것인가&quot;</strong>에 대해 깊이 고민해 볼 수 있었습니다.</p>