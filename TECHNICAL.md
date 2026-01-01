# 🔐 Text Cipher - 기술 문서

> 고전 암호화 알고리즘의 구현 방법에 대한 기술적 설명

---

## 📋 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [시저 암호 구현](#시저-암호-구현)
3. [ROT13 구현](#rot13-구현)
4. [비즈네르 암호 구현](#비즈네르-암호-구현)
5. [치환 암호 구현](#치환-암호-구현)
6. [아트배시 암호 구현](#아트배시-암호-구현)
7. [AES-256 암호화](#aes-256-암호화)
8. [XOR 암호화](#xor-암호화)
9. [인코딩 방식들](#인코딩-방식들)
10. [빈도 분석](#빈도-분석)
11. [게임 로직](#게임-로직)
12. [UI/UX 구현](#uiux-구현)

---

## 프로젝트 개요

### 핵심 개념
고전 암호화 알고리즘들은 대부분 **문자 치환(Substitution)** 방식을 사용합니다. 각 알고리즘은 치환 규칙이 다릅니다.

### 공통 패턴

```javascript
// 모든 암호화 함수의 기본 패턴
function encrypt(text, key) {
    return text.replace(/[a-zA-Z]/g, (char) => {
        // 대소문자 구분을 위한 기준점
        const base = char <= 'Z' ? 65 : 97;
        // 변환 로직
        return transformedChar;
    });
}
```

### 정규표현식 활용

```javascript
// 알파벳만 처리 (대소문자 구분)
text.replace(/[a-zA-Z]/g, callback);

// 영문 대문자만
text.replace(/[A-Z]/g, callback);

// 알파벳이 아닌 문자 제거
key.replace(/[^A-Z]/g, '');
```

---

## 시저 암호 구현

### 알고리즘 원리

시저 암호는 각 문자를 알파벳에서 고정된 수(시프트)만큼 이동시킵니다.

```
알파벳: A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
시프트3: D E F G H I J K L M N O P Q R S T U V W X Y Z A B C
```

### 수학적 표현

```
암호화: E(x) = (x + n) mod 26
복호화: D(x) = (x - n) mod 26
```

여기서 `x`는 문자의 위치(A=0, B=1, ...), `n`은 시프트 값입니다.

### JavaScript 구현

```javascript
caesar: {
    encrypt: (text, shift) => {
        shift = parseInt(shift) || 3;
        return text.replace(/[a-zA-Z]/g, (char) => {
            // 대문자는 65('A'), 소문자는 97('a')를 기준으로
            const base = char <= 'Z' ? 65 : 97;
            
            // 현재 문자의 알파벳 인덱스 (0-25)
            const charIndex = char.charCodeAt(0) - base;
            
            // 시프트 적용 후 26으로 나눈 나머지 (순환)
            const shiftedIndex = (charIndex + shift) % 26;
            
            // 다시 문자로 변환
            return String.fromCharCode(shiftedIndex + base);
        });
    },
    
    decrypt: (text, shift) => {
        shift = parseInt(shift) || 3;
        // 복호화는 (26 - shift)만큼 암호화하는 것과 동일
        return Ciphers.caesar.encrypt(text, 26 - (shift % 26));
    }
}
```

### 단계별 예시

```
입력: "HELLO", 시프트: 3

H (72) → 72 - 65 = 7  → (7 + 3) % 26 = 10 → 10 + 65 = 75 → K
E (69) → 69 - 65 = 4  → (4 + 3) % 26 = 7  → 7 + 65 = 72  → H
L (76) → 76 - 65 = 11 → (11 + 3) % 26 = 14 → 14 + 65 = 79 → O
L (76) → 76 - 65 = 11 → (11 + 3) % 26 = 14 → 14 + 65 = 79 → O
O (79) → 79 - 65 = 14 → (14 + 3) % 26 = 17 → 17 + 65 = 82 → R

결과: "KHOOR"
```

---

## ROT13 구현

### 알고리즘 원리

ROT13은 시저 암호의 특수한 경우로, 시프트 값이 13입니다. 영어 알파벳이 26자이므로 ROT13을 두 번 적용하면 원문으로 돌아옵니다.

```
ROT13(ROT13(x)) = x
```

### JavaScript 구현

```javascript
rot13: {
    encrypt: (text) => Ciphers.caesar.encrypt(text, 13),
    decrypt: (text) => Ciphers.caesar.encrypt(text, 13)
}
```

ROT13은 암호화와 복호화가 동일한 **대칭 암호(symmetric cipher)**입니다.

### 왜 13인가?

```
알파벳 개수: 26
26 / 2 = 13

A → N (13칸 이동)
N → A (13칸 이동, 순환)
```

---

## 비즈네르 암호 구현

### 알고리즘 원리

비즈네르 암호는 키워드를 사용하여 각 문자마다 다른 시프트 값을 적용합니다. 이는 시저 암호보다 훨씬 강력합니다.

```
평문:   A T T A C K A T D A W N
키:     L E M O N L E M O N L E
시프트: 11 4 12 14 13 11 4 12 14 13 11 4
암호문: L X F O P V E F R N H R
```

### 수학적 표현

```
암호화: Ci = (Pi + Ki) mod 26
복호화: Pi = (Ci - Ki + 26) mod 26
```

### JavaScript 구현

```javascript
vigenere: {
    encrypt: (text, key) => {
        // 키 정규화: 대문자로, 알파벳만
        if (!key) key = 'KEY';
        key = key.toUpperCase().replace(/[^A-Z]/g, '');
        if (key.length === 0) key = 'KEY';
        
        let keyIndex = 0;
        return text.replace(/[a-zA-Z]/g, (char) => {
            const base = char <= 'Z' ? 65 : 97;
            
            // 키 문자의 시프트 값 (A=0, B=1, ...)
            const shift = key.charCodeAt(keyIndex % key.length) - 65;
            
            // 다음 알파벳 문자에서 다음 키 문자 사용
            keyIndex++;
            
            return String.fromCharCode(
                (char.charCodeAt(0) - base + shift) % 26 + base
            );
        });
    },
    
    decrypt: (text, key) => {
        if (!key) key = 'KEY';
        key = key.toUpperCase().replace(/[^A-Z]/g, '');
        if (key.length === 0) key = 'KEY';
        
        let keyIndex = 0;
        return text.replace(/[a-zA-Z]/g, (char) => {
            const base = char <= 'Z' ? 65 : 97;
            const shift = key.charCodeAt(keyIndex % key.length) - 65;
            keyIndex++;
            
            // 복호화: 시프트를 빼고, 음수 방지를 위해 +26
            return String.fromCharCode(
                (char.charCodeAt(0) - base - shift + 26) % 26 + base
            );
        });
    }
}
```

### 키 순환

```javascript
// 키가 "KEY"이고 텍스트가 "HELLO WORLD"인 경우
// 알파벳만 카운트하여 키 인덱스 증가

H - K (index 0)
E - E (index 1)
L - Y (index 2)
L - K (index 0, 순환)
O - E (index 1)
  - (공백은 건너뜀)
W - Y (index 2)
O - K (index 0)
...
```

---

## 치환 암호 구현

### 알고리즘 원리

치환 암호는 알파벳의 각 문자를 다른 문자로 일대일 매핑합니다. 키워드를 사용하여 치환 알파벳을 생성합니다.

### 치환 알파벳 생성

```
키워드: KEYWORD

1. 키워드에서 중복 제거: K, E, Y, W, O, R, D
2. 나머지 알파벳 순서대로 추가: A, B, C, F, G, H, I, J, L, M, N, P, Q, S, T, U, V, X, Z

결과: KEYWORDABCFGHIJLMNPQSTUVXZ

원본: ABCDEFGHIJKLMNOPQRSTUVWXYZ
치환: KEYWORDABCFGHIJLMNPQSTUVXZ
```

### JavaScript 구현

```javascript
substitution: {
    encrypt: (text, key) => {
        const alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
        key = (key || '').toUpperCase().replace(/[^A-Z]/g, '');
        
        // 치환 알파벳 생성
        let subAlphabet = '';
        
        // 1. 키워드의 각 문자를 순서대로 추가 (중복 제외)
        for (let c of key) {
            if (!subAlphabet.includes(c)) {
                subAlphabet += c;
            }
        }
        
        // 2. 나머지 알파벳 추가
        for (let c of alphabet) {
            if (!subAlphabet.includes(c)) {
                subAlphabet += c;
            }
        }
        
        // 치환 수행
        return text.replace(/[a-zA-Z]/g, (char) => {
            const upper = char === char.toUpperCase();
            const index = alphabet.indexOf(char.toUpperCase());
            const mapped = subAlphabet[index];
            return upper ? mapped : mapped.toLowerCase();
        });
    },
    
    decrypt: (text, key) => {
        // 복호화: 치환 알파벳에서 원본 알파벳으로 역매핑
        const alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
        key = (key || '').toUpperCase().replace(/[^A-Z]/g, '');
        
        let subAlphabet = '';
        for (let c of key) {
            if (!subAlphabet.includes(c)) subAlphabet += c;
        }
        for (let c of alphabet) {
            if (!subAlphabet.includes(c)) subAlphabet += c;
        }
        
        return text.replace(/[a-zA-Z]/g, (char) => {
            const upper = char === char.toUpperCase();
            // 치환 알파벳에서 위치 찾기
            const index = subAlphabet.indexOf(char.toUpperCase());
            // 원본 알파벳에서 해당 위치 문자 반환
            const mapped = alphabet[index];
            return upper ? mapped : mapped.toLowerCase();
        });
    }
}
```

---

## 아트배시 암호 구현

### 알고리즘 원리

아트배시는 알파벳을 역순으로 치환합니다.

```
원본: A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
치환: Z Y X W V U T S R Q P O N M L K J I H G F E D C B A
```

### 수학적 표현

```
E(x) = 25 - x
D(x) = 25 - x  (동일)
```

### JavaScript 구현

```javascript
atbash: {
    encrypt: (text) => {
        return text.replace(/[a-zA-Z]/g, (char) => {
            const base = char <= 'Z' ? 65 : 97;
            // 25 - (현재 인덱스) = 역순 위치
            return String.fromCharCode(
                base + 25 - (char.charCodeAt(0) - base)
            );
        });
    },
    
    // 아트배시는 대칭 암호
    decrypt: (text) => Ciphers.atbash.encrypt(text)
}
```

### 단계별 예시

```
H: base=65, index=7,  25-7=18  → S
E: base=65, index=4,  25-4=21  → V
L: base=65, index=11, 25-11=14 → O
L: base=65, index=11, 25-11=14 → O
O: base=65, index=14, 25-14=11 → L

HELLO → SVOOL
```

---

## AES-256 암호화

### 알고리즘 원리

AES(Advanced Encryption Standard)는 현대 대칭키 암호화의 표준입니다. 256비트 키를 사용하며, 정부 및 금융권에서 널리 사용됩니다.

### Web Crypto API

브라우저의 `crypto.subtle` API를 사용하여 진정한 암호화를 구현합니다.

```javascript
aes: {
    encrypt: async (text, password) => {
        const encoder = new TextEncoder();
        const data = encoder.encode(text);
        
        // 1. 비밀번호에서 키 유도 (PBKDF2)
        const keyMaterial = await crypto.subtle.importKey(
            'raw', 
            encoder.encode(password), 
            'PBKDF2', 
            false, 
            ['deriveKey']
        );
        
        // 2. 솔트와 반복으로 강력한 키 생성
        const salt = encoder.encode('TextCipherSalt!');
        const aesKey = await crypto.subtle.deriveKey(
            { 
                name: 'PBKDF2', 
                salt, 
                iterations: 100000, 
                hash: 'SHA-256' 
            },
            keyMaterial,
            { name: 'AES-GCM', length: 256 },
            false,
            ['encrypt']
        );
        
        // 3. 랜덤 IV(초기화 벡터) 생성
        const iv = crypto.getRandomValues(new Uint8Array(12));
        
        // 4. 암호화
        const encrypted = await crypto.subtle.encrypt(
            { name: 'AES-GCM', iv },
            aesKey,
            data
        );
        
        // 5. IV + 암호문을 합쳐서 Base64로 인코딩
        const combined = new Uint8Array(iv.length + encrypted.byteLength);
        combined.set(iv);
        combined.set(new Uint8Array(encrypted), iv.length);
        
        return btoa(String.fromCharCode(...combined));
    }
}
```

### 키 생성 및 클립보드 복사

```javascript
function generateAESKey() {
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*';
    let key = '';
    for (let i = 0; i < 32; i++) {
        key += chars.charAt(Math.floor(Math.random() * chars.length));
    }
    return key;
}

function copyKeyToClipboard(key, type) {
    navigator.clipboard.writeText(key).then(() => {
        showToast('🔑', `${type} 키가 클립보드에 복사되었습니다!`);
        document.getElementById('cipherKey').value = key;
    });
}
```

---

## XOR 암호화

### 알고리즘 원리

XOR(Exclusive OR)은 비트 연산을 이용한 암호화입니다. 같은 키로 두 번 XOR하면 원문이 복원됩니다.

```
평문:  01001000 (H)
키:    01010011 (S)
XOR:   00011011 (암호문)

암호문: 00011011
키:     01010011 (S)
XOR:    01001000 (H) - 원문 복원!
```

### JavaScript 구현

```javascript
xor: {
    encrypt: (text, key) => {
        if (!key) {
            key = generateRandomString(16);
            copyKeyToClipboard(key, 'XOR');
        }
        
        let result = '';
        for (let i = 0; i < text.length; i++) {
            // 각 문자를 키의 해당 위치 문자와 XOR
            result += String.fromCharCode(
                text.charCodeAt(i) ^ key.charCodeAt(i % key.length)
            );
        }
        
        // 결과를 Base64로 인코딩 (특수문자 처리)
        return btoa(result);
    },
    
    decrypt: (text, key) => {
        const decoded = atob(text);
        let result = '';
        for (let i = 0; i < decoded.length; i++) {
            result += String.fromCharCode(
                decoded.charCodeAt(i) ^ key.charCodeAt(i % key.length)
            );
        }
        return result;
    }
}
```

### One-Time Pad

키 길이가 평문과 같고 완전히 랜덤하면, XOR 암호는 **이론적으로 완벽한 보안**(정보이론적 안전성)을 제공합니다.

---

## 인코딩 방식들

### Base64

바이너리 데이터를 64개의 안전한 ASCII 문자로 변환합니다.

```javascript
base64: {
    encrypt: (text) => btoa(unescape(encodeURIComponent(text))),
    decrypt: (text) => decodeURIComponent(escape(atob(text)))
}
```

### Hex (16진수)

```javascript
hex: {
    encrypt: (text) => {
        return [...text]
            .map(c => c.charCodeAt(0).toString(16).padStart(2, '0'))
            .join(' ');
    },
    decrypt: (text) => {
        return text.split(/\s+/)
            .map(h => String.fromCharCode(parseInt(h, 16)))
            .join('');
    }
}
```

### Binary (2진수)

```javascript
binary: {
    encrypt: (text) => {
        return [...text]
            .map(c => c.charCodeAt(0).toString(2).padStart(8, '0'))
            .join(' ');
    }
}
```

### 제로폭 문자 숨김

보이지 않는 유니코드 문자를 사용하여 메시지를 숨깁니다.

```javascript
zerowidth: {
    encrypt: (text) => {
        // Zero-width space (0), Zero-width non-joiner (1)
        const zwc = { '0': '\u200B', '1': '\u200C' };
        
        // 텍스트를 2진수로 변환
        const binary = [...text]
            .map(c => c.charCodeAt(0).toString(2).padStart(8, '0'))
            .join('');
        
        // 2진수를 제로폭 문자로 변환
        const hidden = binary.split('').map(b => zwc[b]).join('');
        
        // 일반 텍스트 사이에 숨김
        return '일반 텍스트 ' + hidden + ' 여기에 비밀이 있습니다';
    }
}
```

---

## 빈도 분석

### 알고리즘 원리

문자 빈도 분석은 암호 해독의 기본 기법입니다. 영어에서 각 문자의 출현 빈도는 일정한 패턴을 보입니다.

### 영어 문자 빈도

```
E: 12.7%  T: 9.1%   A: 8.2%   O: 7.5%   I: 7.0%
N: 6.7%   S: 6.3%   H: 6.1%   R: 6.0%   D: 4.3%
...
```

### JavaScript 구현

```javascript
function analyzeText() {
    const text = document.getElementById('analysisInput').value.toUpperCase();
    const freq = {};
    let total = 0;

    // 각 알파벳의 빈도 계산
    for (let char of text) {
        if (char >= 'A' && char <= 'Z') {
            freq[char] = (freq[char] || 0) + 1;
            total++;
        }
    }

    // 시각화를 위한 차트 생성
    const alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
    let maxFreq = Math.max(...alphabet.split('').map(c => freq[c] || 0));

    // 막대 그래프 렌더링
    alphabet.split('').forEach(letter => {
        const count = freq[letter] || 0;
        const height = maxFreq > 0 ? (count / maxFreq) * 60 : 0;
        const percent = total > 0 ? ((count / total) * 100).toFixed(1) : 0;
        // 막대 생성...
    });
}
```

### 활용 방법

1. 암호문의 빈도 분석 수행
2. 가장 빈번한 문자가 'E'일 가능성 높음
3. 시프트 값 추정: (암호문 최빈 문자) - E = 시프트
4. 단어 패턴 확인으로 검증

---

## 게임 로직

### 퍼즐 생성

```javascript
function generatePuzzle() {
    // 가능한 문구 목록
    const phrases = ['HELLO WORLD', 'SECRET MESSAGE', ...];
    
    // 레벨에 따른 암호 종류
    const cipherTypes = [
        { type: 'caesar', getKey: () => Math.floor(Math.random() * 25) + 1 },
        { type: 'rot13', getKey: () => null },
        { type: 'atbash', getKey: () => null }
    ];
    
    // 높은 레벨에서 더 어려운 암호 추가
    if (gameState.level >= 3) {
        cipherTypes.push({
            type: 'vigenere',
            getKey: () => ['KEY', 'CODE', 'CIPHER'][Math.random() * 3 | 0]
        });
    }
    
    // 랜덤 선택
    const phrase = phrases[Math.random() * phrases.length | 0];
    const cipher = cipherTypes[Math.random() * cipherTypes.length | 0];
    const key = cipher.getKey();
    
    // 암호화
    const encrypted = Ciphers[cipher.type].encrypt(phrase, key);
    
    gameState.currentPuzzle = {
        original: phrase,
        encrypted: encrypted,
        type: cipher.type,
        key: key
    };
}
```

### 점수 시스템

```javascript
function checkAnswer() {
    const input = document.getElementById('gameInput').value.toUpperCase().trim();
    const answer = gameState.currentPuzzle.original;

    if (input === answer) {
        gameState.correct++;
        gameState.streak++;
        
        // 점수 계산: 기본 점수 + 레벨 보너스 + 연속 정답 보너스
        gameState.score += 10 * gameState.level + gameState.streak * 2;
        
        // 최고 스트릭 갱신
        if (gameState.streak > gameState.bestStreak) {
            gameState.bestStreak = gameState.streak;
        }
        
        // 레벨업 (5문제마다)
        if (gameState.correct % 5 === 0) {
            gameState.level++;
        }
        
        // 다음 문제 생성
        setTimeout(generatePuzzle, 1500);
    } else {
        gameState.streak = 0; // 연속 정답 초기화
    }
}
```

### 힌트 시스템

```javascript
function showHint() {
    gameState.hintsUsed++;
    const puzzle = gameState.currentPuzzle;
    
    let hint = `암호 방식: ${puzzle.typeName}`;
    
    // 첫 번째 힌트: 키/시프트 값
    if (puzzle.type === 'caesar') {
        hint += ` | 시프트: ${puzzle.key}`;
    } else if (puzzle.type === 'vigenere') {
        hint += ` | 키: ${puzzle.key}`;
    }
    
    // 두 번째 힌트: 첫 글자
    if (gameState.hintsUsed >= 2) {
        hint += ` | 첫 글자: ${puzzle.original[0]}`;
    }
    
    document.getElementById('gameHint').textContent = hint;
}
```

---

## UI/UX 구현

### Matrix 배경 애니메이션

```javascript
function initMatrixBg() {
    const bg = document.getElementById('matrixBg');
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@#$%^&*';
    
    for (let i = 0; i < 20; i++) {
        const column = document.createElement('div');
        column.className = 'matrix-column';
        
        // 랜덤 위치
        column.style.left = `${Math.random() * 100}%`;
        
        // 랜덤 속도
        column.style.animationDuration = `${10 + Math.random() * 20}s`;
        column.style.animationDelay = `${Math.random() * 10}s`;
        
        // 랜덤 문자열 생성
        let text = '';
        for (let j = 0; j < 50; j++) {
            text += chars[Math.floor(Math.random() * chars.length)] + '<br>';
        }
        column.innerHTML = text;
        
        bg.appendChild(column);
    }
}
```

### CSS 애니메이션

```css
@keyframes matrixFall {
    0% { transform: translateY(0); }
    100% { transform: translateY(200vh); }
}

.matrix-column {
    position: absolute;
    top: -100%;
    animation: matrixFall linear infinite;
}
```

### 알파벳 매핑 시각화

```javascript
function updateAlphabetDisplay() {
    const type = cipherType.value;
    const alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
    
    if (type === 'caesar') {
        const shift = parseInt(shiftSlider.value);
        
        display.innerHTML = alphabet.split('').map(letter => {
            const mapped = String.fromCharCode(
                (letter.charCodeAt(0) - 65 + shift) % 26 + 65
            );
            
            return `
                <div class="alphabet-letter">
                    <span class="original">${letter}</span>
                    <span class="mapped">${mapped}</span>
                </div>
            `;
        }).join('');
    }
}
```

---

## 결론

### 핵심 기술 요약

| 기능 | 핵심 로직 |
|------|----------|
| 시저 암호 | `(char + shift) % 26` |
| ROT13 | 시저 암호 (shift=13) |
| 비즈네르 | 키 순환 + 시저 암호 |
| 치환 암호 | 키워드 기반 매핑 테이블 |
| 아트배시 | `25 - index` |
| 빈도 분석 | 문자 카운팅 + 정규화 |

### 보안 관점

고전 암호들은 현대 기준으로 **안전하지 않습니다**:
- 시저/ROT13: 26가지 경우의 수만 시도하면 해독
- 치환 암호: 빈도 분석으로 해독 가능
- 비즈네르: 카시스키 테스트로 키 길이 추정 가능

실제 보안이 필요하면 AES, RSA 등 현대 암호화 알고리즘을 사용하세요.

---

*마지막 업데이트: 2026년 1월*

