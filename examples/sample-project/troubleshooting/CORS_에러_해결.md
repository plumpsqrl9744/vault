# CORS 에러 해결

#cors #express #해결완료

## 증상

프론트(localhost:5173)에서 백엔드(localhost:3000) API 호출 시 CORS 에러 발생.

```
Access to fetch at 'http://localhost:3000/api/users' from origin 'http://localhost:5173' 
has been blocked by CORS policy
```

## 원인 (Why)

Express 서버에 CORS 미들웨어가 설정되지 않았음.
브라우저는 서로 다른 origin 간 요청을 기본 차단한다.

## ���결 (Do)

```javascript
// server.js
const cors = require('cors');

app.use(cors({
  origin: 'http://localhost:5173',
  credentials: true
}));
```

## 참고
- 프로덕션에서는 origin을 실제 도메인으로 제한할 것
