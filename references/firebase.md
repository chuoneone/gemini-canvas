# Firebase 初始化（Canvas 專用，僅需要雲端存檔／歷史紀錄時才加）

只用 Canvas 注入的全域變數，並以 `typeof` 保護。寫法取自實際運作中的 Canvas 工具。

```js
let db, auth, firestoreMod, localAppId, currentUser;

async function initFirebase() {
    try {
        const appMod = await import("https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js");
        const authMod = await import("https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js");
        firestoreMod = await import("https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js");

        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        localAppId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const app = appMod.initializeApp(firebaseConfig);
        auth = authMod.getAuth(app);
        db = firestoreMod.getFirestore(app);

        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
            await authMod.signInWithCustomToken(auth, __initial_auth_token);
        } else {
            await authMod.signInAnonymously(auth);
        }
        authMod.onAuthStateChanged(auth, (u) => { currentUser = u; });
    } catch (e) {
        console.error("Firebase init failed", e);
    }
}
```

注意：集合路徑要使用 `localAppId` 區隔，例如 `artifacts/${localAppId}/users/${currentUser.uid}/…`，以符合 Canvas 的資料規則。請依你的 Canvas 環境實際規則調整路徑。
