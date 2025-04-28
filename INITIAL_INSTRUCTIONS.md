# Οδηγίες Υλοποίησης Υποδομής AWS για Consul Democracy (Ανάπτυξη & Επίδειξη - Χειροκίνητος Κωδικός RDS)

**Στόχος:** Να δημιουργηθεί μια ευέλικτη υποδομή στο AWS για τη φιλοξενία της εφαρμογής Consul Democracy. Η υποδομή θα χρησιμοποιείται κυρίως για ανάπτυξη (3-4 developers) και περιστασιακές επιδείξεις (~30 χρήστες). Θα αποτελείται από μία EC2 instance και μία RDS instance, οι οποίες θα μπορούν να αλλάζουν μέγεθος (resize) ανάλογα με τις ανάγκες, με δυνατότητα παύσης/εκκίνησης της EC2 για εξοικονόμηση κόστους.

**Συνοδευτικό Υλικό:** Παρακαλώ ανατρέξτε και στο αρχικό PDF τεκμηρίωσης του Consul Democracy για τις απαιτήσεις της εφαρμογής.

## Βασική Αρχιτεκτονική:

*   **EC2 Instance:** Application Server (Ubuntu 24.04, Nginx, Puma, Ruby, Node.js, Consul dependencies). Θα αλλάζει μέγεθος μεταξύ `t3.large` (normal) και `t3.large` (demo).
*   **RDS Instance:** PostgreSQL Database Server (PostgreSQL 16.x). Θα αλλάζει μέγεθος μεταξύ `db.t3.small` (normal) και `db.t3.large` (demo).
*   **Networking:** Κατάλληλο VPC, Custom Security Groups, Elastic IP για την EC2.
*   **Deployment:** Capistrano.
*   **Extras:** Βασικό Monitoring (CloudWatch), Custom Domain (Route 53), SSL (Let's Encrypt/Certbot).

## Απαιτούμενα Πριν την Έναρξη:

*   **Πρόσβαση σε λογαριασμό AWS:** Με επαρκή δικαιώματα (IAM Permissions) για δημιουργία EC2, RDS, Security Groups, Elastic IPs, Route 53 records, CloudWatch Alarms.
*   **Ζεύγος κλειδιών SSH (SSH Key Pair):** Καταχωρημένο στο AWS (π.χ., `consul-democracy-key`). Το ιδιωτικό κλειδί (`.pem` αρχείο) πρέπει να είναι διαθέσιμο στον υπολογιστή που θα εκτελέσει το Ansible.
*   **Δημόσιες IP διευθύνσεις των developers:** Για πρόσβαση SSH στο EC2 Security Group.
*   **Domain Name:** (π.χ., `yourdomain.com`) για τη ρύθμιση του custom domain και SSL.
*   **Ισχυρός Κωδικός Βάσης Δεδομένων RDS:** Αποθηκευμένος με ασφάλεια. Συνιστάται η χρήση Ansible Vault για την αποθήκευση αυτού του κωδικού κατά την εκτέλεση του playbook.
*   **Εγκατάσταση Ansible:** Το Ansible πρέπει να είναι εγκατεστημένο στον τοπικό υπολογιστή (ή όπου θα εκτελεστεί το playbook). (Οδηγίες: <https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html>)
*   **Python 3 στην EC2:** Βεβαιωθείτε ότι η επιλεγμένη Ubuntu 24.04 AMI περιλαμβάνει Python 3 (συνήθως το περιλαμβάνει). Το Ansible το χρειάζεται για να τρέξει στον remote server.

## Αναθεωρημένα Βήματα Υλοποίησης:

### Βήμα 1: Δικτύωση & Ασφάλεια (VPC & Security Groups) - (Παραμένει ίδιο)

*   **VPC:** Χρησιμοποιήστε το Default VPC ή το `consul-democracy-vpc` που δημιουργήσατε.
*   **Security Group για RDS (`consul-db-sg`)**
    *   Δημιουργία (Έγινε).
    *   Inbound Rule: Type: PostgreSQL (TCP 5432), Source: Το Security Group ID του `consul-app-sg` (Θα το ορίσουμε αφού δημιουργηθεί η EC2 και το SG της).
*   **Security Group για EC2 (`consul-app-sg`)**
    *   Δημιουργία (Έγινε).
    *   Inbound Rules:
        *   Type: SSH (TCP 22), Source: ΜΟΝΟ οι IPs των developers.
        *   Type: HTTP (TCP 80), Source: Anywhere (0.0.0.0/0) (Για αρχική ρύθμιση και Let's Encrypt).
        *   Type: HTTPS (TCP 443), Source: Anywhere (0.0.0.0/0).
    *   Outbound Rule: Type: All traffic, Destination: Anywhere (0.0.0.0/0) (Έγινε).

### Βήμα 2: Δημιουργία Βάσης Δεδομένων (RDS Instance) - (Παραμένει ίδιο)

1.  Μεταβείτε στην κονσόλα AWS RDS.
2.  Πατήστε "Create database".
3.  Engine: PostgreSQL, Version: 16.x.
4.  Templates: Dev/Test.
5.  Settings:
    *   DB instance identifier: `consul-democracy-db`.
    *   Master username: `consuladmin`.
    *   Master password: Ορίστε τον ισχυρό κωδικό (και αποθηκεύστε τον με ασφάλεια, ιδανικά σε Ansible Vault).
6.  Instance class: `db.t3.small` (Αρχική ρύθμιση).
7.  Storage: gp3, 30 GiB.
8.  Availability: Do not create a standby instance (Single-AZ).
9.  Connectivity:
    *   VPC: Επιλέξτε το σωστό VPC.
    *   Public access: No.
    *   VPC security group: Επιλέξτε "Choose existing" και διαλέξτε το `consul-db-sg`.
10. Additional configuration:
    *   Initial database name: `consul_democracy_production`.
11. Backup: Enable (π.χ., 7 days).
12. Πατήστε "Create database".
13. **Σημειώστε:** Το Endpoint address (hostname) και το Port (5432) της βάσης. Θα τα χρειαστούμε για το Ansible.

### Βήμα 3: Δημιουργία Application Server (EC2 Instance) - (Παραμένει σχεδόν ίδιο)

1.  Μεταβείτε στην κονσόλα AWS EC2.
2.  Πατήστε "Launch instances".
3.  Name: `consul-democracy-app`.
4.  AMI: Ubuntu Server 24.04 LTS (x86).
5.  Instance type: `t3.large` (Αρχική ρύθμιση).
6.  Key pair: Επιλέξτε το `consul-democracy-key`.
7.  Network settings:
    *   VPC: Κατάλληλο VPC.
    *   Subnet: Επιλέξτε μια public subnet.
    *   Auto-assign public IP: Enable (Προσωρινά, μέχρι να συνδέσουμε Elastic IP).
    *   Firewall (security groups): Επιλέξτε "Select existing security group" και διαλέξτε το `consul-app-sg`.
8.  Configure storage: 1x 50 GiB gp3 Root volume.
9.  Advanced details: Αφήστε το "IAM instance profile" κενό.
10. Πατήστε "Launch instance".
11. **Δέσμευση & Σύνδεση Elastic IP:**
    *   Μεταβείτε στο "Elastic IPs".
    *   Πατήστε "Allocate Elastic IP address".
    *   Συνδέστε (Associate) το νέο Elastic IP με την `consul-democracy-app` instance.
    *   **Σημειώστε:** Αυτό το Elastic IP. Θα το χρειαστούμε για το Ansible και το DNS.
12. **Ενημέρωση Security Group Βάσης:**
    *   Επιστρέψτε στο Security Group `consul-db-sg`.
    *   Επεξεργαστείτε τον Inbound Rule για τη θύρα 5432.
    *   Στο πεδίο "Source", επιλέξτε το Security Group ID του `consul-app-sg`.
    *   Αποθηκεύστε τις αλλαγές.

### Βήμα 4: Ρύθμιση Λογισμικού στην EC2 μέσω Ansible (Αντικαθιστά το παλιό Βήμα 4)

Στον τοπικό σας υπολογιστή (ή όπου έχετε Ansible):

1.  **Εγκατάσταση Ansible:** Βεβαιωθείτε ότι είναι εγκατεστημένο.
2.  **Κλωνοποίηση Installer Repo:**
    ```bash
    git clone https://github.com/consuldemocracy/installer.git
    cd installer
    ```
3.  **Δημιουργία Inventory File (`hosts`)**
    *   Αντιγράψτε το παράδειγμα: `cp hosts.example hosts`
    *   Επεξεργαστείτε το αρχείο `hosts`:
        *   Αντικαταστήστε το `remote-server-ip-address` με το Elastic IP της EC2 instance σας.
        *   Βεβαιωθείτε ότι το `ansible_user` είναι σωστό (πιθανόν `ubuntu` αρχικά, αν το Ansible θα δημιουργήσει τον `deploy` χρήστη, ή `deploy` αν τον δημιουργήσετε χειροκίνητα με `sudo`). Το playbook μπορεί να χρειαστεί να τρέξει αρχικά ως χρήστης με `sudo` χωρίς password ή ως `root` για να δημιουργήσει τον `deploy` χρήστη. Εναλλακτικά, μπορείτε να ορίσετε `ansible_user=ubuntu ansible_become=yes` αν ο `ubuntu` έχει `sudo` χωρίς password. Ας υποθέσουμε ότι το playbook χειρίζεται τη δημιουργία του `deploy` χρήστη.
        *   Ορίστε το `ansible_ssh_private_key_file` στη διαδρομή του `.pem` κλειδιού σας (π.χ., `ansible_ssh_private_key_file=/path/to/your/consul-democracy-key.pem`).
4.  **Ρύθμιση Μεταβλητών Ansible:**
    *   Επεξεργαστείτε το αρχείο `group_vars/all` (ή χρησιμοποιήστε `--extra-vars` ή Ansible Vault).
    *   Οπωσδήποτε ορίστε τις μεταβλητές για την RDS:
        *   `database_hostname`: Το Endpoint address της RDS.
        *   `database_name`: `consul_democracy_production`
        *   `database_user`: `consuladmin`
        *   `database_password`: Ο κωδικός της RDS. **Χρησιμοποιήστε Ansible Vault γι' αυτό!** (Δημιουργήστε ένα `group_vars/all/vault.yml` και κρυπτογραφήστε το με `ansible-vault encrypt group_vars/all/vault.yml`).
    *   Ορίστε τις μεταβλητές για Domain/SSL:
        *   `domain`: `yourdomain.com` (Αντικαταστήστε με το δικό σας domain).
        *   `letsencrypt_email`: `your-email@example.com` (Για τις ειδοποιήσεις του Let's Encrypt).
    *   Προαιρετικά: Ελέγξτε και προσαρμόστε άλλες μεταβλητές στο `group_vars/all` αν χρειάζεται (π.χ., `timezone`, `deploy_user`, `deploy_group`). Για Ubuntu, το `deploy_group: sudo` είναι πιθανώς σωστό.
5.  **Εκτέλεση του Ansible Playbook (`app.yml`)**
    *   Τρέξτε την εντολή από τον φάκελο `installer`:
        ```bash
        # Αν χρησιμοποιείτε Vault, θα σας ζητηθεί ο κωδικός
        ansible-playbook -v app.yml -i hosts --ask-vault-pass
        ```
    *   (Σημείωση: Το `app.yml` είναι το κύριο playbook του installer repo).
    *   Το Ansible θα συνδεθεί στην EC2, θα εγκαταστήσει όλα τα dependencies (Ruby, Node, Nginx, Puma, libs), θα ρυθμίσει το Nginx, το Puma service, τον χρήστη `deploy`, τη δομή φακέλων του Capistrano, θα ρυθμίσει τα `database.yml` και `secrets.yml` (χρησιμοποιώντας τις μεταβλητές που δώσατε), θα εγκαταστήσει το Certbot και θα πάρει SSL πιστοποιητικό.

### Βήμα 5: Ρύθμιση Deployment Εφαρμογής (Capistrano) - (Τροποποιημένο)

1.  **Τοπική Ρύθμιση Capistrano:**
    *   Στον κώδικα της εφαρμογής Consul Democracy (όχι στο `installer` repo), ρυθμίστε τα αρχεία του Capistrano (`Capfile`, `config/deploy.rb`, `config/deploy/production.rb`).
    *   Στο `config/deploy/production.rb`:
        *   Ορίστε το `server` με το Elastic IP.
        *   Ορίστε τον `user` σε `deploy` (ή όποιον χρήστη ορίσατε/δημιούργησε το Ansible).
        *   Ρόλοι (`:app`, `:web`, `:db` - αν και το `:db` τρέχει στην RDS, ο ρόλος μπορεί να χρειάζεται για migrations).
    *   **Διαχείριση Secrets:** Το Ansible θα πρέπει να έχει ήδη τοποθετήσει τα `database.yml` και `secrets.yml` (με τα credentials της RDS) στον φάκελο `shared/config` στον server. Το `deploy.rb` του Capistrano (που πιθανόν ρυθμίστηκε από το Ansible) θα πρέπει να περιλαμβάνει εντολές για `linked_files` που να συνδέουν αυτά τα αρχεία κατά το deployment. Δεν χρειάζεται να διαχειριστείτε τα credentials απευθείας μέσα στο Capistrano.
    *   **SSH Key για Capistrano:** Βεβαιωθείτε ότι το δημόσιο κλειδί SSH του χρήστη που θα τρέχει `cap deploy` είναι στο `~/.ssh/authorized_keys` του χρήστη `deploy` στην EC2. Το Ansible playbook μπορεί να το έχει κάνει ήδη αν χρησιμοποιήσατε τη μεταβλητή `ssh_public_key_path`.
2.  **Δοκιμαστική Ανάπτυξη:**
    ```bash
    cap production deploy:check # Ελέγχει βασικές ρυθμίσεις
    cap production deploy # Κάνει deploy τον κώδικα της εφαρμογής
    ```
    *   Το Capistrano θα κλωνοποιήσει τον κώδικα, θα τρέξει `bundle install`, `assets:precompile`, `db:migrate` (στην RDS) και θα επανεκκινήσει τον Puma server.

### Βήμα 6: Διαδικασίες Κόστους & Κλιμάκωσης - (Παραμένει ίδιο)

*   **Stop/Start EC2:** Μέσω της κονσόλας AWS EC2 για εξοικονόμηση όταν δεν χρησιμοποιείται.
*   **Resize (Upscale/Downscale):**
    *   **EC2:** Stop -> Change Instance Type -> Start.
    *   **RDS:** Modify -> Change DB instance class (μπορεί να γίνει και εν λειτουργία με μικρό downtime, ειδικά αν ήταν Multi-AZ, αλλά εδώ είναι Single-AZ οπότε θα υπάρξει downtime).

### Βήμα 7: Ρύθμιση Monitoring (CloudWatch - Βασικό) - (Παραμένει ίδιο)

*   **CloudWatch Agent:** Εγκαταστήστε και ρυθμίστε τον Agent στην EC2 χειροκίνητα (ή προσαρμόστε/επεκτείνετε το Ansible playbook) για metrics όπως Memory % Used.
*   **CloudWatch Alarms:** Δημιουργήστε alarms για EC2 CPUUtilization, Memory % Used (από Agent), RDS CPUUtilization, RDS FreeableMemory, RDS DiskQueueDepth κ.λπ.

### Βήμα 8: Ρύθμιση Custom Domain & SSL - (Κυρίως μέσω Ansible)

*   **Route 53:** Δημιουργήστε χειροκίνητα ένα A Record στο Hosted Zone του domain σας που να δείχνει στο Elastic IP της EC2.
*   **Nginx & SSL:** Το Ansible (Βήμα 4) θα έχει ρυθμίσει το Nginx να ακούει στο domain που ορίσατε και θα έχει χρησιμοποιήσει το Certbot για να εγκαταστήσει το SSL πιστοποιητικό, κάνοντας αυτόματη ανακατεύθυνση από HTTP σε HTTPS.

## Τελικοί Έλεγχοι:

*   Επισκεφθείτε το `https://yourdomain.com`. Η εφαρμογή πρέπει να φορτώνει σωστά μέσω HTTPS.
*   Δοκιμάστε βασικές λειτουργίες που αλληλεπιδρούν με τη βάση (π.χ., εγγραφή, σύνδεση αν είναι δυνατόν).
*   Επαληθεύστε ότι το deployment με `cap production deploy` λειτουργεί.
*   Δοκιμάστε τις διαδικασίες Stop/Start/Resize (προαιρετικά).
*   Ελέγξτε τα βασικά metrics στο CloudWatch.

---

Αυτές οι οδηγίες παρέχουν ένα πλήρες πλαίσιο για την υλοποίηση. Ο προγραμματιστής θα πρέπει να προσαρμόσει τις εντολές και τις διαδρομές αρχείων ανάλογα με τις ιδιαιτερότητες της εγκατάστασης του Consul Democracy και του περιβάλλοντος Ubuntu.

---
My server is up. Has been created and running. I make changes and run this command cap production deploy. Help me make this edits
 % cap production deploy
 ssh -i ~/.ssh/consul-democracy-key.pem ubuntu@3.65.166.91
