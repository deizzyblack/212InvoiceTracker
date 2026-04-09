# Login başarısızlığı (200 OK olmasına rağmen)

Bu repo içinde frontend/backend kodu bulunmadığı için doğrudan kod üzerinde patch uygulanamadı. Aşağıdaki notlar, verdiğin semptomlara göre en olası nedeni ve kontrol adımlarını içerir.

## En olası kök neden

`/auth/login` endpoint'i **200 OK + token** dönüyor, fakat frontend tarafı başarılı login için farklı bir JSON şekli bekliyor.

Tipik uyumsuzluk:

- Backend dönüşü: `{ "access_token": "...", "token_type": "bearer" }`
- Frontend beklentisi: `{ "token": "...", "user": { ... } }` veya `response.user`

Bu durumda API çağrısı teknik olarak başarılı olsa da, `auth-context` içinde `if (!data.user) throw new Error(...)` gibi bir guard çalışır ve UI'da `Login failed. Please try again.` görünür.

## Hızlı doğrulama

1. Browser DevTools > Network > login request > Response JSON'u aç.
2. `auth-context` içinde login sonrası hangi alanların zorunlu kontrol edildiğine bak (`user`, `token`, `accessToken` vb.).
3. İki tarafın alan adlarını birebir eşleştir.

## Uyumlandırma seçenekleri

### Seçenek A — Frontend'i backend'e uydur

`authAPI.login` dönüşünü normalize et:

```ts
const raw = await authAPI.login(email, password);
const normalized = {
  token: raw.token ?? raw.access_token,
  user: raw.user ?? raw.data?.user ?? null,
};
```

Ardından auth-context'i `normalized` üstünden ilerlet.

### Seçenek B — Backend'i frontend'e uydur

Login response'u frontend'in beklediği sabit şemaya getir:

```json
{
  "token": "<jwt>",
  "user": {
    "id": 1,
    "email": "admin@demo.com",
    "role": "admin"
  }
}
```

## İkincil kontrol listesi

- `Content-Type: application/json` doğru mu?
- `Authorization` header formatı `Bearer <token>` mı?
- CORS ve `credentials` ayarı frontend isteği ile uyumlu mu?
- Token localStorage'a yazıldıktan sonra route guard hemen eski state'i mi okuyor?
- `isAuthenticated` sadece `user` varlığına mı bakıyor, yoksa token doğrulaması da var mı?

## Sonuç

Belirti setine göre problem büyük olasılıkla **kimlik doğrulama response şeması uyumsuzluğu**. Yani backend başarılı, fakat frontend başarılı kabul koşulunu sağlayamıyor.
