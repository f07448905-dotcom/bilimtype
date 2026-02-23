# Bilim Type: Firebase арқылы жаңа логикаға көшу нұсқаулығы

## 1) Архитектура (ұсыныс)
- **Frontend (Web):** `index.html` (оқушы/мұғалім кіру, теру), `progress.html` (үлкен экрандағы live-board).
- **Firebase Auth:** Role-based login (Admin, Teacher, Student, Parent).
- **Cloud Firestore:**
  - `classes/{classId}`
  - `students/{studentId}`
  - `texts/{textId}`
  - `progressSessions/{sessionId}` (live progress)
  - `dailyChampions/{date}`
- **Cloud Functions:** күндік чемпионды есептеу, Google Sheets-ке синк.
- **Google Sheets:** studentId, score, accuracy, progress, time, timestamp сақтау.

## 2) Firestore data model

### classes
```json
{
  "id": "3A",
  "grade": 3,
  "language": "kk",
  "teacherUid": "uid_teacher_1"
}
```

### students
```json
{
  "id": "S001",
  "fullName": "Аян Сәрсен",
  "classId": "3A",
  "language": "kk",
  "parentUid": "uid_parent_1",
  "authUid": "uid_student_1"
}
```

### texts
```json
{
  "title": "Көктем",
  "content": "...",
  "grade": 3,
  "language": "kk",
  "isActive": true
}
```

### progressSessions
```json
{
  "studentId": "S001",
  "classId": "3A",
  "progress": 72,
  "accuracy": 91,
  "score": 6,
  "status": "running",
  "updatedAt": "serverTimestamp"
}
```

## 3) Қадам-қадамымен Firebase орнату
1. Firebase Console → New project.
2. Web app тіркеу, config алу.
3. Auth қосу: Email/Password.
4. Firestore қосу (Production mode).
5. Hosting қосу (`firebase init hosting`).
6. Functions қосу (`firebase init functions`).

## 4) Рөлдік жүйе (Admin/Teacher/Student/Parent)
1. Auth-тан кейін custom claims беру:
   - admin: толық қолжетімділік.
   - teacher: өз сыныбының progress/students.
   - student: тек өз дерегі.
   - parent: тек өз баласының дерегі.
2. Cloud Function (`setUserRole`) арқылы claim орнату.

## 5) Firestore security rules (үлгі)
```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function role() { return request.auth.token.role; }

    match /students/{studentId} {
      allow read: if role() == 'admin'
        || (role() == 'teacher')
        || (role() == 'student' && request.auth.uid == resource.data.authUid)
        || (role() == 'parent' && request.auth.uid == resource.data.parentUid);
      allow write: if role() == 'admin' || role() == 'teacher';
    }

    match /progressSessions/{id} {
      allow read: if role() in ['admin', 'teacher']
        || (role() == 'student' && request.auth.uid == resource.data.authUid)
        || (role() == 'parent' && request.auth.uid == resource.data.parentUid);
      allow create, update: if role() in ['admin', 'teacher', 'student'];
    }
  }
}
```

## 6) Логика талаптарын іске асыру
- **Теру кезінде шығу бұғаттау:** `beforeunload` + UI `lockExit`.
- **Аяқтау батырмасы:** міндетті confirm.
- **3-сынып сүзгісі:** `grade === 3` мәтіндері ғана.
- **Тілді авто таңдау:** class→language map (`3A=kk`, `3M=ru`).
- **Оқушыны тізімнен таңдау:** free text алып тастау, тек `students` collection.
- **Live board:** `progressSessions` realtime listener, progress descending sort.

## 7) Daily Champion жүйесі
- Күн соңында scheduled Cloud Function:
  - бүгінгі сессияларды алады.
  - score + accuracy формуласы бойынша рейтинг жасайды.
  - `dailyChampions/{yyyy-mm-dd}` құжатына сақтайды.
- Teacher/Admin бетте «Күн чемпионы» карточкасын шығару.

## 8) Google Sheets интеграциясы
1. Service Account JSON кілтін Functions secrets-ке салыңыз.
2. Cloud Function (`onProgressSaved`) триггері:
   - Firestore `progressSessions` жаңарса, Sheets-ке append row.
3. Әр оқушы ID-мен жеке диаграмма:
   - Pivot/Chart: x=timestamp, y=score/accuracy/progress.

## 9) Миграция реті
1. Алдымен `students`, `classes`, `texts` импорт.
2. Сосын Auth account + role claim.
3. Frontend-ті Firebase SDK-ға ауыстыру.
4. Live-board пен champion функцияларын қосу.
5. Соңында Google Sheets синк.
