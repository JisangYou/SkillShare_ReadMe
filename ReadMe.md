# SkillShare 사용 기술 stack(client, server 구분?)

## 사용 Library

### Layout 및 뷰 관련 library

- cardview
- bottom navigation
- togglebutton
- flexboxLayout

### network 관련 library

- retrofit2
    - gsonconverter, adapter-rxjava

- okhttp

### 동영상 관련 library

- exo player

### LogIn 관련 library

- google login

### notification 관련 library

- google cloud messaging

### image loading 관련 library

- glide

### Reactive 관련 library

- rxandroid2
- rxbinding2

### 기타

- lambda

## Application structure

- 사진첨부

## DB Modeling

- 사진 첨부

## Application analysis


## Splash Activity

### _구조 및 기능_

- 최초 app 화면으로, 앱의 진입점을 담당하는 Activity

### _issue_

- 자동로그인 관련 처리
- gif파일 화면세팅 처리

### _How to solve?_

> __자동로그인__

- 정상적으로 SignUp,SignIn이 이루어지면 해당 Token값이 Preference에 저장이 되고, 나중에 app을 켤때는 __Preference에 저장되어 있는 값을 다음과 같이 추출하는 과정__을 통해 자동으로 로그인을 할 수 있는 정보가 세팅이 된다.

```Java
String token = PreferenceUtil.getStringValue(this, ConstantUtil.AUTHORIZATION_FLAG);
```

- 이하 세팅된 token 값을 활용한 서버 통신 과정.

```Java
if(!ValidationUtil.isEmpty(token)) {
            RetrofitHelper.createApi(UserService.class)
                    .getMyInfo(token)
                    .subscribeOn(Schedulers.io())
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(
                            (UserResponse userResponse) -> {
                                if(ConstantUtil.SUCCESS.equals(userResponse.getResult())) {
                                    StateUtil.getInstance().setUserInstance(userResponse.getData());
                                    Intent intent = new Intent(SplashActivity.this, MainActivity.class);
                                    intent.setAction(ConstantUtil.SIGN_IN_SUCCESS);
                                    startActivity(intent);
                                    finish();
                                } else {
                                    Toast.makeText(this, userResponse.getMessage(), Toast.LENGTH_LONG).show();
                                }
                            }, (Throwable error) -> {

                            }
                    );
        } else {
            setContentView(R.layout.activity_splash);
```

> __Glide를 활용한 gif파일 화면 세팅__

- glide library를 활용하면 쉽게 gif파일을 세팅할 수 있다.
- gif 프레임 추출 > 프레임 사이즈 최적화 > 프레임 저장

```java
  
            Glide.with(this)
                    .load(R.raw.splash)
                    .apply(RequestOptions.diskCacheStrategyOf(DiskCacheStrategy.RESOURCE)) // decoded resource 를 Disk Cache 저장소에 저장해둔다
                    .into((ImageView) findViewById(R.id.splash_image));
```

## SignIn Activity

### _구조 및 기능_

- 일반 로그인과 google 로그인을 하는 로직이 담긴 Activity

### _issue_

- 로그인 시 자동로그인을 위한 token 값 저장
- 로그인에 필요한 정규식과 정규식 조건 충족시 Rxbinding을 활용한 뷰 처리
- google logIn

### _How to solve?_

> __자동로그인__

- 아래의 로직은 preference를 사용해 값을 저장하는 로직으로써, 확장성을 고려해서 PreferenceUtil이라는 클래스를 따로 만들어서 관리
 
```Java
 PreferenceUtil.setStringValue(this, ConstantUtil.AUTHORIZATION_FLAG, response.getToken());
```

> __로그인 관련 정규식__

- util성 클래스를 따로 만들어, SignIn Activity에서 이를 활용해서 처리

```Java
public class ValidationUtil {
    private static final Pattern EMAIL_REGEX = Pattern.compile("^[_0-9a-zA-Z-]+@[0-9a-zA-Z-]+(.[_0-9a-zA-Z-]+)*$");
    private static final Pattern PW_REGEX = Pattern.compile("^[a-zA-Z0-9!@.#$%^&*?_~]{8,16}$"); // 8 - 16 자리

    public static boolean isEmpty(String any) {
        if(any == null || "".equals(any))
            return true;
        else
            return false;
    }

    public static boolean isValidEmailAddress(String email_address) {
        return EMAIL_REGEX.matcher(email_address).matches();
    }

    public static boolean isValidPassword(String password) {
        return PW_REGEX.matcher(password).matches();
    }
}
```

> __Rxbinding을 활용한 뷰 컨트롤__

- rxbinding을 하면 기존의 textWatcher를 활용한 것보다 코드가 간결하다.

```Java
Observable<CharSequence> o1 = RxTextView.textChanges(editTextEmail);
        Observable<CharSequence> o2 = RxTextView.textChanges(editTextPassword);

        // Observable 1 과 Observable 2 를 결합해서 결과값을 반환
        Observable.combineLatest(o1, o2,
                (e, p) -> ValidationUtil.isValidEmailAddress(e.toString()) &&
                        ValidationUtil.isValidPassword(p.toString()))
        .subscribe(
                (validity) -> {
                    if(validity) {
                        buttonSignIn.setTextColor(Color.BLACK);
                    } else {
                        buttonSignIn.setTextColor(Color.LTGRAY);
                    }
                }
        );
```

## SignUp Activity

### _구조 및 기능_

- 회원가입 과정을 통해 서버에 해당 정보를 등록하는 기능을 가진 Activity


### _issue_

- 회원가입 시 id, password 조건을 위한 정규식 세팅
- 서버에 정보를 등록하는 networking


### _How to solve?_

> 정규식 세팅 및 서버에 등록

- 회원가입 전에 email, password 등의 정보들을 등록하는 과정은 위에서 언급했던 ValidationUtil을 사용해서 처리한다.
- 정의해놓은 조건들을 모두 충족하게 되면, 서버에 등록하는 networking이 진행된다. 

```Java
 findViewById(R.id.button_sign_up).setOnClickListener(
                v -> {
                    // progress bar
                    String email = editTextEmailAddress.getText().toString();
                    String password = editTextPassword.getText().toString();
                    String first_name =  editTextFirstName.getText().toString();
                    String last_name = editTextLastName.getText().toString();
                    if( !ValidationUtil.isValidEmailAddress(email) ) {
                        // email 형식이 틀렸습니다
                    } else if( !ValidationUtil.isValidPassword(password) ) {
                        // password 를 올바르게 입력해주세요 8 - 16 자
                    } else if ( ValidationUtil.isEmpty(first_name) || ValidationUtil.isEmpty(last_name) ) {
                        // 이름을 입력해주세요
                    } else {
                        // request body setting
                        SignUpRequestBody signUpRequestBody = new SignUpRequestBody();
                        signUpRequestBody.setEmail(email);
                        signUpRequestBody.setPassword(password);
                        name = first_name+" "+last_name;
                        signUpRequestBody.setName(first_name+" "+last_name);

                        // retrofit
                        RetrofitHelper.createApi(UserService.class).signUp(signUpRequestBody)
                                .subscribeOn(Schedulers.io())
                                .observeOn(AndroidSchedulers.mainThread())
                                .subscribe(this::handleResponse, this::handleError);
                    }
                }
        );
```
- 아래는 retrofit2을 사용하기 위한 interface양식이다.
- Body를 POST방식으로 서버에 보내주면, 서버에서 받아서 Body에 담긴 정보들을 User모델에 세팅하는 과정이 실행된다.

```Java
@POST("user/sign-up")
    Observable<UserResponse> signUp(@Body SignUpRequestBody body);
```

## Main Activity

### _구조 및 기능_

- App의 기둥이 되는 Activity로써, 6개의 fragment와 Bottom Navigation 등으로 구성되어 있다.
- __검색기능__ 및 __frament들을 관리__하는 기능이 있다.
- 로그인 및 로그아웃 등의 상황에 따른 로직을 관리하여 각 fragment에 필요한 데이터를 전달한다. 

### _issue_

- NavigationBar item에 전환에 따른 fragment 세팅
- googleGcm 세팅
- 로그인, 로그아웃 시에 따른 상황별 로직

### _How to solve?_

> __NavigationBar + fragment 세팅__

- NavigationBar 아이템에 따른 fragment의 add 및 replace를 컨트롤하는 로직은 아래와 같다.
- (참고) 총 6 개의 프래그먼트가 있고, 회원(비회원)이라는 조건에 따라 프로필을 담당하는 Mefragment(OffLineFragment)는 둘 중 하나만 NavigationBar에 세팅이 된다.
- (참고2) 총 여섯개의 case가 있으나, 로직은 아래와 같기때문에 생략한다. 

```Java
  bottomNavigation.setOnTabSelectedListener((int position, boolean wasSelected) -> {
            switch (position) {
                case 0:
                    if (wasSelected) {
                        Log.d("JUWONLEE", "0 was selected");
                        return false;
                    } else {
                        if(homeFragment == null) {
                            Log.d("JUWONLEE", "0 null");
                            homeFragment = new HomeFragment();

                            fragments.add(homeFragment);

                            getSupportFragmentManager()
                                    .beginTransaction()
                                    .add(R.id.fragment_container, homeFragment)
                                    .commit();

                            stack = fragments.indexOf(homeFragment);
                        } else if(fragments.contains(homeFragment)) {
                            Log.d("JUWONLEE", "0 contains");
                            getSupportFragmentManager()
                                    .beginTransaction()
                                    .hide(fragments.get(stack))
                                    .show(homeFragment)
                                    .commit();

                            stack = fragments.indexOf(homeFragment);
                        } else {
                            Log.d("JUWONLEE", "0 else");
                            getSupportFragmentManager()
                                    .beginTransaction()
                                    .hide(fragments.get(stack))
                                    .show(homeFragment)
                                    .commit();

                            stack = fragments.indexOf(homeFragment);
                        }
                        return true;
                    }
```

> __google GCM 사용을 위한 BroadCastReceiver과 Service__

- 댓글을 달거나, 좋아요 등의 이벤트가 발생했을 때을 위한 notification관련 로직들이다.
- 4대 컴포넌트 중 BradCastReceiver과 Service 기능이 담긴 로직들을 사용을 했다.

```Java
   // Broad Cast Receiver
    private void registerReceiver() {
        LocalBroadcastManager localBroadcastManager = LocalBroadcastManager.getInstance(this);
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(ConstantUtil.RECEIVED);
        localBroadcastManager.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                if (intent.getAction().equals(ConstantUtil.RECEIVED)) {
                    // notification 받으면 bottom navigation view 에 noti 달기
                    bottomNavigation.setNotification(" ", 1);
                }
            }
        }, intentFilter);
    }
```
```Java
 private void startRegisterService() {
        if(stateUtil.getState()) {
            Intent intent = new Intent(this, RegistrationIntentService.class);
            intent.putExtra("USER_ID", stateUtil.getUserInstance().get_id());
            startService(intent);
        }
    }
```
- 위의 기능을 사용하기위해 google에서 제공하는 상세 로직들은 project내 GCM package에 있으며, 해당 package 관련 핵심 내용은 아래와 같다.

> (참고) GCM 관련 핵심요약
- MyGcmListenerService은 서버에서 푸시를 보내면 수신하여 처리하는 부분
- MyInstanceIDListenerService 각 디바이스마다 토큰이라고 하는 고유 값이 생성되는데 이러한 토큰의 신규 발급, 순환, 업데이트가 발생될때 처리하는 부분
- RegistrationIntentService 토큰을 생성하는 부분
- 참고 블로그 [구글gcm](https://developers.google.com/cloud-messaging/)

> (참고) localBroadCast 장점
- 현재 프로세스 안에서만 유효한 Broadcast
- Global Broadcast 에 비해 훨씬 시스템 부하가 적다. 
- 액티비티 내부의 객체간에 상호 의존성을 낮추어 깔끔한 프로그램 구조를 만들 수 있다는 점이 장점이다.
- 출처: [localbraodCast](https://m.blog.naver.com/PostView.nhn?blogId=rosaria1113&logNo=205338870&proxyReferer=https%3A%2F%2Fwww.google.co.kr%2F)


## HomeFragment(in MainActivity)

### _구조 및 기능_

- ![home](https://user-images.githubusercontent.com/31605792/35291153-c4844b86-00af-11e8-9dfa-f0cf2a9af9a3.png)
- 위와 같이 아이템들이 바둑판 형식으로 구성되어 있는 구조

### _issue_

- 바둑판 형식으로 구성되어 있는 레이아웃구조
- 동적으로 추가될 수 있는 아이템들을 고려한 로직 

### _How to solve?_

> __중첩된 recyclerView와 <List<Map<String,List<T>>>방식을 통해 중첩된 RecyclerView 데이터세팅__

- 수직 방향으로 설정한 recyclerView 안에 아이템으로써 수평방향으로 설정한 recyclerView를 세팅
- 편의상 필요한 부분의 로직만 가지고 옴.

- 아래와 같이 holder에 리싸이클러뷰를 만들어 주고, onBindViewHolder에 수평방향의 recyclerView를 만들어줌으로써 처리했다.
- outer recyclerView안에 아이템으로써의 inner recycler뷰를 map<String, List<T>>가 같은 형태로 으로 제어했다.
- 그 이유는 추후에 inner recycler뷰가 동적으로 추가될 경우가 있고, 그 안에 데이터를 넣기위해서는 map형태로 키값을 찾아 그 안에 데이터를 넣게 해주기 위함이다.   
- inner RecyclerView는 일반적인 방법과 같으므로 생략한다. 

```Java
public class HomeRecyclerViewAdapter extends RecyclerView.Adapter<HomeRecyclerViewAdapter.Holder> {
    List<Map<String, List<Class>>> classes;

    @Override
    public void onBindViewHolder(Holder holder, int position) {
        Map<String, List<Class>> map = classes.get(position);

        for (String title : map.keySet()) {
            holder.textViewTitle.setText(title);
            holder.generalRecyclerView.setLayoutManager(new LinearLayoutManager(context, LinearLayoutManager.HORIZONTAL, false));
            holder.generalRecyclerView.setAdapter(new GeneralRecyclerViewAdapter(context, map.get(title)));
        }
    }


    class Holder extends RecyclerView.ViewHolder {

        TextView textViewTitle, buttonSeeAll;
        RecyclerView generalRecyclerView;

        public Holder(View view) {
            super(view);
            textViewTitle = view.findViewById(R.id.text_view_title);
            generalRecyclerView = view.findViewById(R.id.general_recycler_view);
            buttonSeeAll = view.findViewById(R.id.button_see_all);
            
        }
    }

}

```


## classActivity

### _구조 및 기능_

- ![class](https://user-images.githubusercontent.com/31605792/35291171-d9447ca8-00af-11e8-90c7-90ad6a3c9f34.png)
- 수업컨텐츠를 클릭했을때 보여지는 Activity이다.
- skillShare에서 제공하는 핵심 컨텐츠
- 동영상을 시청할 수 있는 기능이 있다.
- 탭레이아웃 + 프래그먼트 구조를 지니고 있다.

### _issue_

- Activity와 Fragment간 통신
- exoLibrary를 활용한 동영상 시청하기 기능

### _How to solve?_

> __bundle을 통해 통신__

- tabLayout과 ViewPager 그리고 3개의 프래그먼트를 사용해서 레이아웃을 구성했다.
- bundle을 통해 해당 class의 id값만 전달한다. 
- 각 fragment에서는 해당 id값으로 필요한 데이터를 서버에서 불러오는 로직을 구현했다. 

```Java
 private void setTabLayout() {
        tabLayout.addTab(tabLayout.newTab().setText("Lessons"));
        tabLayout.addTab(tabLayout.newTab().setText("About"));
        tabLayout.addTab(tabLayout.newTab().setText("Discussion"));

        tabLayout.addOnTabSelectedListener(new TabLayout.ViewPagerOnTabSelectedListener(tabPager));
    }

    private void setTabPager() {
        List<Fragment> fragmentList = new ArrayList<>();

        Bundle bundle = new Bundle();
        bundle.putString(ConstantUtil.ID_FLAG, classId); // 선택된 클래스의 id값만 bundle에 담았고, 각 프래그먼트는 그 id값으로 서버와 통신할 것이다.

        lessonsfragment = new LessonsFragment();
        lessonsfragment.setArguments(bundle); // bundle값 전달
        aboutfragment = new AboutFragment();
        aboutfragment.setArguments(bundle);// bundle값 전달
        discussionsfragment = new DiscussionsFragment();
        discussionsfragment.setArguments(bundle);// bundle값 전달

        fragmentList.add(lessonsfragment);
        fragmentList.add(aboutfragment);
        fragmentList.add(discussionsfragment);

        tabPager.setAdapter(new FragmentAdapter(getSupportFragmentManager(), fragmentList));
        tabPager.addOnPageChangeListener(new TabLayout.TabLayoutOnPageChangeListener(tabLayout));
    }
```

> __exoPlayer library__ 

- 세팅과정
    1. ExoPlayer를 프로젝트에 종속성으로 추가한다.
    2. SimpleExoPlayer인스턴스를 만든다 .
    3. 플레이어를 뷰에 연결한다 (비디오 출력 및 사용자 입력 용).
    4. MediaSource재생할 플레이어를 준비한다.
    5. 완료되면 플레이어를 넣는다.

- 커스터마이징
    - Renderer- Renderer라이브러리가 제공하는 기본 구현에서 지원되지 않는 미디어 유형을 처리 하는 사용자 정의를 구현할 수 있습니다 .
    - TrackSelector- 커스텀을 구현 TrackSelector하면 앱 개발자가 a MediaSource로 노출 된 트랙을 사용 가능한 각각의 트랙 으로 선택 하는 방식을 변경할 수 있습니다 Renderer.
    - LoadControl- 사용자 정의를 구현 LoadControl하면 앱 개발자가 플레이어의 버퍼링 변경할 수 있다. 
    - Extractor- 라이브러리에서 현재 지원되지 않는 컨테이너 형식을 지원해야하는 경우 사용자 정의 Extractor클래스를 구현하는 것을 고려한 다음 ExtractorMediaSource해당 유형의 미디어를 재생 하는 데 함께 사용할 수 있습니다.
    - MediaSource- 사용자 정의 MediaSource클래스를 구현하는 것은 사용자 정의 방법으로 렌더러에 피드 할 미디어 샘플을 얻고 싶거나 사용자 정의 MediaSource합성 동작 을 구현하려는 경우에 적합 할 수 있습니다.
    - DataSource- ExoPlayer의 업스트림 패키지에는 DataSource다양한 사용 사례에 대한 여러 가지 구현이 이미 포함되어 있습니다 . DataSource사용자 지정 프로토콜을 통해, 사용자 지정 HTTP 스택을 사용하여 또는 사용자 지정 영구 캐시에서와 같이 다른 방식으로 데이터를로드하기 위해 자체 클래스 를 구현할 수 있습니다 .

- [exoplayerGuide](http://google.github.io/ExoPlayer/guide.html)

- [전체 코드]()

```java
private void initiatePlayer() {
        userAgent = Util.getUserAgent(this, "Skill Share");

        // view
        simpleExoPlayerView = new SimpleExoPlayerView(this);
        simpleExoPlayerView = findViewById(R.id.simple_exo_player_view);
        simpleExoPlayerView.requestFocus(); // ( ? )
        simpleExoPlayerView.setUseArtwork(true);
        simpleExoPlayerView.setUseController(true); //Set media controller
        simpleExoPlayerView.setControllerHideOnTouch(false);
        simpleExoPlayerView.showController();

        // renders [ 4, 5 ]
        DefaultRenderersFactory renderersFactory = new DefaultRenderersFactory(this, null);

        // track selector [ 5 ]
        TrackSelection.Factory adaptiveTrackSelectionFactory =
                new AdaptiveTrackSelection.Factory(BANDWIDTH_METER);
        trackSelector = new DefaultTrackSelector(adaptiveTrackSelectionFactory);

        player = ExoPlayerFactory.newSimpleInstance(renderersFactory, trackSelector);
        player.addListener(new PlayerEventListener());

        // binding
        simpleExoPlayerView.setPlayer(player);
        player.setPlayWhenReady(false);

        // dash
        // data source factory
        mediaDataSourceFactory = buildDataSourceFactory(true);

        // [ 1, 2, 3 ( ? ) ]
        mediaSource = buildMediaSource(Uri.parse(URL));

        if(resumePosition > 0)
            player.seekTo(resumePosition);
        player.prepare(mediaSource);
    }
```
- 아래는 비디오가 실행되면서 생길 수 있는 다양한 상황에서 원활한 재생을 위해 custome한 method이다.

```Java
 private void saveResumePosition() {// 현재 재생된 시간까지 저장
        
        resumePosition = player.getContentPosition();
        resumePosition = (resumePosition > 0) ? resumePosition : 0;
    }


    private void releasePlayer() { // player 객체가 있으면, 저장된 시간부터 시작
        if(player != null) {
            saveResumePosition();
            player.release();
            player = null;
            trackSelector = null;
        }
    }

    @Override                           
    protected void onPause() {  // 생명주기를 활용해 잠깐 정지시에 멈춤
        player.setPlayWhenReady(false);
        super.onPause();
    }

    @Override
    protected void onDestroy() {// 앱이 꺼졌을때도 상태저장
        releasePlayer();
        super.onDestroy();
    }
```


## LessonFragment(in ClassActivity)

### _구조 및 기능_

- ![lesson](https://user-images.githubusercontent.com/31605792/35291171-d9447ca8-00af-11e8-90c7-90ad6a3c9f34.png)
- content제작자(tutor)에 대한 정보를 나타내는 Activity

### _issue_

- Follow, Unfollow 토글버튼 클릭 이벤트 처리

### _How to solve?_

> 여러 상황을 고려해서 처리

```Java

if (StateUtil.getInstance().getState()) {   // 로그인을 하고 들어온 경우
            List<Following> followings = StateUtil.getInstance().getUserInstance().getFollowing();
            for (Following following : followings) {            // user가 가지고 있는 following 모델안에 데이터를 싹 돌면서,
                if (following.getUserId().equals(tutorId)) {   // userId와 tutorId가 같은게 있다면, 이미 follwing을 했다는 의미이므로
                    buttonFollow.setChecked(true);              // 체크가 된 상태로 뷰가 생성된다.
                    buttonFollow.setTextColor(getResources().getColor(R.color.white));
                    break;
                }
            }

            buttonFollow.setOnCheckedChangeListener((buttonView, isChecked) -> {
                if (isChecked) {    //toggle버튼이 체크가 되면, followers는 한명 증가
                    buttonView.setTextColor(getResources().getColor(R.color.white));
                    followers++;
                    toggleFollow();
                } else { // 다시 toggle버튼이 체크가 되면 followers 원상 복귀
                    buttonView.setTextColor(getResources().getColor(R.color.IcActive));
                    followers--;
                    toggleFollow();
                }

                textViewFollowersCount.setText(followers + " Followers");
            });

        } else { // 로그인이 안되어있을때는 popup으로 로그인창이 뜬다.
            buttonFollow.setOnCheckedChangeListener((buttonView, isChecked) ->
                    {
                        buttonView.setChecked(false);
                        DialogUtil.showSignDialog(context);
                    }
            );
        }
```


## AboutFragment(in ClassActivity)

### _구조 및 기능_

- ![about](https://user-images.githubusercontent.com/31605792/35291194-eeb2c04a-00af-11e8-9506-6d04b8fae529.png)
- 해당 컨텐츠가 가지고 있는 프로젝트, 리뷰, 구독자, 연관된 클래스를 정보를 나타내는 Activity

### _issue_

- seeAll(더보기)관련 처리

### _How to solve?_

> constantUtil을 활용하여 더보기 처리

- 앱내에 많은 더보기 기능을 처리하기 위해 constantUtil을 활용했다.
- 더보기 클릭 이벤트가 일어날때 해당 키값으로 constantUtil에 정의 해놓은 값을 사용한다.
- 예를 들어, 더보기 클릭 이벤트에 다음과 같이 정의해놓고, 
```java
 textViewReviewSeeAll.setOnClickListener(view -> {
                Intent intent = new Intent(context, SeeAllActivity.class);
                intent.putExtra(ConstantUtil.SEE_ALL_FLAG, ConstantUtil.REVIEW_ITEM);
                intent.putExtra(ConstantUtil.ID_FLAG, classId);
                intent.putParcelableArrayListExtra(ConstantUtil.REVIEWS_FLAG, (ArrayList<? extends Parcelable>) reviews);
                context.startActivity(intent);
            });
```
- SeeAll Activity
```Java
// ...중략

    intent = getIntent(); 
    //이와 같이 seeAll 클릭이벤트시 전달한 intent를 한 액티비티에서 일괄적으로 전달받는다. 
    type = intent.getStringExtra(ConstantUtil.SEE_ALL_FLAG); 
    // 전달시에 key값은 ConstantUtil.SEE_ALL_FLAG으로 통일하고, value는 이벤트가 일어나는 곳의 특징을 할 수 있는 naming을 해서 전달해주기 때문에 아래와 같은 로직으로 처리 할 수 있다.

//...중략


 switch (type) { 
     // type에는 위에 주석으로 남겨놓은 것 처럼, 해당 이벤트가 일어난 곳의 특징을 알 수 있는 값이 담겨 있을 것이므로, case문을 자동으로 해당 값에 맞는 곳에 들어가 로직이 실행된다.
            case ConstantUtil.CLASS_ITEM:

                break;
            case ConstantUtil.REVIEW_ITEM:
                // 예를 들어, 위와 같은 경우 ConstantUtil.REVIEW_ITEM를 값을 보냈기에, type은 ConstantUtil.REVIEW_ITEM으로 정의가 되어 있을 것이므로, 이곳에 작성한 로직대로 동작을 할 것이다.
                ...생략

                // Intent를 다음과 같이 ConstantUtil에 명시해놓은 키값으로 받아서 사용한다.
                List<Review> reviews = intent.getParcelableArrayListExtra(ConstantUtil.REVIEWS_FLAG);

                int size = reviews.size();
                int likeCount = 0;
                for(Review review : reviews) {
                    if("like".equals(review.getLikeOrNot()))
                        likeCount++;
                }

                //...중략

                recyclerView.setAdapter(new ReviewSeeAllRecyclerViewAdapter(this, reviews));
                
                break;  // break은 항상 잊지 않는다.
```


## discussionFragment

### _구조 및 기능_

### _issue_

### _How to solve?_



## GroupFragment 

### _구조 및 기능_

![group](https://user-images.githubusercontent.com/31605792/35291397-81bf61f4-00b0-11e8-8897-20ee18747653.png)

### _issue_

### _How to solve?_