
![[Screenshot 2025-11-30 at 14.25.47.png]]
![[Screenshot 2025-11-30 at 14.26.29.png]]
![[Screenshot 2025-11-30 at 14.27.20.png]]
![[Screenshot 2025-11-30 at 14.29.39.png]]

![[Screenshot 2025-11-30 at 14.51.41.png]]

![[Screenshot 2025-11-30 at 14.51.22.png]]
![[Screenshot 2025-11-30 at 14.52.35.png]]

![[Screenshot 2025-11-30 at 14.53.42.png]]

![[Screenshot 2025-11-30 at 14.53.26.png]]
![[Screenshot 2025-11-30 at 14.54.28.png]]
![[Screenshot 2025-11-30 at 14.56.57.png]]
![[Screenshot 2025-11-30 at 14.57.42.png]]
![[Screenshot 2025-11-30 at 15.14.26.png]]![[Screenshot 2025-11-30 at 15.19.32.png]]
![[Screenshot 2025-11-30 at 15.21.36.png]]
![[Screenshot 2025-11-30 at 15.21.58.png]]
![[Screenshot 2025-11-30 at 15.22.23.png]]
![[Screenshot 2025-11-30 at 15.22.54.png]]
![[Screenshot 2025-11-30 at 15.24.03.png]]
![[Screenshot 2025-11-30 at 15.35.08.png]]
![[Screenshot 2025-11-30 at 15.36.21.png]]
![[Screenshot 2025-11-30 at 15.36.54.png]]
![[Screenshot 2025-11-30 at 15.38.46.png]]
![[Screenshot 2025-11-30 at 15.39.03.png]]
![[Screenshot 2025-11-30 at 15.40.47.png]]
![[Screenshot 2025-11-30 at 15.41.18.png]]
![[Screenshot 2025-11-30 at 15.41.45.png]]
![[Screenshot 2025-11-30 at 15.46.53.png]]
![[Screenshot 2025-11-30 at 15.47.54.png]]
# 🔐 **SSH Authentication Flow (GitHub / Bitbucket)**

## **1️⃣ You generate keys on your computer**

You create:

- **Private key** → stays on your machine
    
- **Public key** → can be shared safely
    

Example:

`id_rsa         (private) id_rsa.pub     (public)`

---

## **2️⃣ You copy the PUBLIC key to GitHub/Bitbucket**

You upload only:

`id_rsa.pub`

GitHub saves that key and says:

> “If someone can prove they own the matching private key, I will trust them.”

---

## **3️⃣ You run a Git command**

Example:

`git push`

VS Code → Git → SSH → Contacts GitHub.

---

## **4️⃣ GitHub sends a challenge to your computer**

GitHub says:

> “Prove you have the **PRIVATE key** matching this public key.”

It sends an encrypted random message (a challenge).

---

## **5️⃣ Your computer decrypts the challenge using PRIVATE key**

Your SSH agent does:

- Receives challenge
    
- Decrypts it using **private key**
    
- Sends back the answer
    

**Private key never leaves your machine.**

---

## **6️⃣ GitHub verifies the answer**

GitHub checks:

- Does the answer match?
    
- Does this correspond to the public key in the account?
    

If yes:

> “Authentication SUCCESS. You are who you say you are.”

---

## **7️⃣ GitHub allows the push/pull**

Now Git operation continues:

- `git pull` downloads your repo
    
- `git push` uploads your commits
    

---

# 🧠 **Visualization of the FLOW**

`(Your Machine)                      (GitHub / Bitbucket) ----------------------------------------------------------- Generate keys Private key ----- stays -----> Public key -------------------> Saved in account  git push ----------------------> SSH connection start                                   "Prove you have private key" <---- Encrypted challenge ------- Decrypt challenge with private key ------ Answer challenge -------->                                   ✔ Verified = OK                                   Allow git operation`

---

# 🎉 You are authenticated **WITHOUT** ever sending any password

- No password across the network
    
- No password stored in Git
    
- No password reused
    
- Only cryptographic proof
![[Screenshot 2025-11-30 at 15.52.04.png]]
![[Screenshot 2025-11-30 at 15.53.12.png]]
![[Screenshot 2025-11-30 at 15.54.14.png]]

![[Screenshot 2025-11-30 at 15.54.03.png]]![[Screenshot 2025-11-30 at 16.01.48.png]]
![[Screenshot 2025-11-30 at 16.02.16.png]]
![[Screenshot 2025-11-30 at 16.04.20.png]]
![[Screenshot 2025-11-30 at 16.05.20.png]]
