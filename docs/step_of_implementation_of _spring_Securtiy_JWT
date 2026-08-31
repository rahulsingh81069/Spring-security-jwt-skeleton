Poori Kahani — Step by Step (Yaad Rakhne Ke Liye)

Step 1 — Dependencies: spring-boot-starter-security + JWT library (jjwt-*) add karo. Why: Security framework aur token banane/verify karne ke tools chahiye.

Step 2 — Entity mein password aur role field: User entity mein yeh do fields add karo. Why: Login ke liye password chahiye, authorization ke liye role chahiye.

Step 3 — PasswordEncoder Bean (SecurityConfig mein): BCryptPasswordEncoder ka bean banao. Why: Password kabhi plain text mein store nahi karte — hash karke store karte hain, aur yeh Bean isliye banaya taaki Spring khud isko manage kare aur kahi bhi @Autowired se mil jaye (Dependency Injection — tight coupling se bachne ke liye).

Step 4 — Signup/Create User mein Password Hash Karo: passwordEncoder.encode(rawPassword) call karo save karne se pehle. Why: Database mein sirf hash jana chahiye, original password kabhi nahi.

Step 5 — JwtUtil Class: Token generate karne ka method (generateToken) aur verify karne ka method (isTokenValid, extractEmail). Why: Yeh "wristband banane wali machine" hai — login successful hone pe wristband (token) deta hai, aur baad mein wristband check karta hai.

Step 6 — AuthController with /login: Email-password leke, database se user dhoondo, passwordEncoder.matches(rawPassword, hashedPassword) se verify karo, match ho toh JwtUtil.generateToken() call karke token do. Why: Yeh "entry gate" hai jaha se wristband milta hai.

Step 7 — JwtAuthFilter: Har request ke Authorization: Bearer <token> header ko check karo, token valid ho toh Spring Security ko batao "yeh authenticated hai" (SecurityContextHolder.setAuthentication(...)). Why: Yeh "security guard" hai jo har request pe wristband check karta hai, bina wristband ke andar jaane nahi deta.

Step 8 — SecurityFilterChain (SecurityConfig mein): csrf.disable(), STATELESS session, kaunse paths permitAll() hain (login/signup) aur kaunse authenticated() chahiye, aur addFilterBefore(jwtAuthFilter, ...) se apna guard sabse pehle lagao. Why: Yeh "building ka rule-book" hai — kaunse gate free hain, kaunse locked hain, aur guard kaha khada hoga.

Golden order jo hamesha yaad rakhna: Encode → Login verifies → Token generate → Filter checks every future request → SecurityFilterChain decides which paths need that check.
