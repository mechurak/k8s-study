# 11. Identity / Keycloak

CKA 시험 범위는 아니지만 **실무에서 앱 인증·SSO(Single Sign-On, 한 번 로그인으로 여러 앱 통과)의 중심**이 되는 Keycloak을 정리한다. 특히 **사내 AD(Active Directory) 연동**과, 내가 만드는 앱이 받는 **토큰(claim)** 의 큰 그림에 집중.

> ArgoCD 등 GitOps 생태계 도구는 [`10_ecosystem-gitops`](../10_ecosystem-gitops/)에. 여기는 **신원/인증(IdP, Identity Provider — 신원 제공자)** 만 다룬다.
> 앱 앞단에서 이 인증을 강제하는 oauth2-proxy는 [`04_services-networking/oauth2-proxy.md`](../04_services-networking/oauth2-proxy.md)에 있다(네트워킹 영역). **Keycloak 안쪽**(realm·federation)이 여기.

## 다루는 내용

### 큰 그림 — realm · client · 토큰 → [concepts.md](./concepts.md)
- **🗺️ 3개 경계, 2번의 매핑** — AD → Keycloak → 내 앱으로 정보가 흐르는 전체 그림
- **Realm** — 격리된 보안 영역(테넌트), master realm vs 업무용 realm
- **Client** — Keycloak을 쓰는 앱 하나(confidential/public, redirect URI, scope)
- **로그인 흐름** — Authorization Code Flow (비번은 앱이 안 받음)
- **앱이 받는 것** — ID/Access/Refresh 토큰, JWT(JSON Web Token) claim 예시(`preferred_username`·`email`·`groups`·`roles`)
- **Protocol Mapper** — 토큰에 무엇을 담을지(경계 ②), 그룹 vs 롤, 토큰 검증(JWKS, JSON Web Key Set)

### AD 연동 — User Federation / LDAP(Lightweight Directory Access Protocol) → [ad-federation.md](./ad-federation.md)
- **🔗 전체 사슬** — AD 그룹(`memberOf`)이 토큰 `groups` claim까지 가는 5단계
- **인증 위임** — 비밀번호는 AD에만, Keycloak은 LDAP bind로 검증만
- **LDAP Provider** — 연결(LDAPS, LDAP over TLS)·검색(Users DN(Distinguished Name)·`sAMAccountName`·`objectGUID`)
- **Edit Mode & Sync** — READ_ONLY/import, full vs changed sync
- **LDAP Mappers** — 속성·그룹·롤·MSAD(Microsoft Active Directory) account-control 매퍼(경계 ①)
- **검증·트러블슈팅** — "그룹이 비어 있음" 등 증상→원인→해결 표

### (추후) 더 볼 것
- Keycloak을 k8s에 배포(Operator/Helm) + 운영(HA(High Availability, 고가용성)·DB·백업)
- Identity Brokering(외부 OIDC(OpenID Connect)/SAML(Security Assertion Markup Language) IdP 연동), 소셜 로그인
- Fine-grained Authorization(Keycloak Authorization Services)

## 참고

- [Keycloak 공식 문서](https://www.keycloak.org/documentation)
- [Keycloak — Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- 관련 → [04 oauth2-proxy.md](../04_services-networking/oauth2-proxy.md) · [04 external-access.md](../04_services-networking/external-access.md)
