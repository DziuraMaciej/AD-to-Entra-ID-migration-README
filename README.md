# Entra ID Profile Migrator

**Entra ID Profile Migrator** to narzędzie desktopowe (Windows Forms, .NET 10, `win-x64`) służące do migracji **In-Place (Zero-Touch)** profili użytkowników Windows (domenowych Active Directory lub lokalnych) na konta chmurowe **Microsoft Entra ID (Azure AD)** — bez kopiowania plików i bez tworzenia nowego katalogu w `C:\Users`.

Aplikacja przeprowadza administratora przez cały proces: od lokalnego wypięcia komputera z domeny Active Directory, przez dołączenie maszyny do Entra ID, aż po podmianę uprawnień ACL, translację gałęzi rejestru (`NTUSER.DAT`, `UsrClass.dat`), rekonfigurację zmapowanych dysków sieciowych (DFS/SMB) oraz przygotowanie pierwszego logowania.

---

## Spis treści

1. [Główne możliwości](#główne-możliwości)
2. [Wymagania systemowe](#wymagania-systemowe)
3. [Architektura i struktura projektu](#architektura-i-struktura-projektu)
4. [Przebieg działania krok po kroku](#przebieg-działania-krok-po-kroku)
   - [Etap 0: Lokalne wypięcie z domeny Active Directory](#etap-0-lokalne-wypięcie-z-domeny-active-directory)
   - [Etap 1: Wybór profilu źródłowego](#etap-1-wybór-profilu-źródłowego)
   - [Etap 2: Dołączenie do Entra ID i wskazanie konta docelowego](#etap-2-dołączenie-do-entra-id-i-wskazanie-konta-docelowego)
   - [Etap 3: Migracja profilu In-Place (Faza I)](#etap-3-migracja-profilu-in-place-faza-i)
   - [Etap 4: Czyszczenie po restarcie (Faza II)](#etap-4-czyszczenie-po-restarcie-faza-ii)
5. [Szczegóły techniczne kluczowych mechanizmów](#szczegóły-techniczne-kluczowych-mechanizmów)
6. [Logowanie i diagnostyka](#logowanie-i-diagnostyka)
7. [Kompilacja i publikacja](#kompilacja-i-publikacja)

---

## Główne możliwości

- **Automatyczna detekcja środowiska AD / WORKGROUP:** Przy starcie aplikacja sprawdza za pomocą natywnego API `NetGetJoinInformation`, czy komputer należy do domeny Active Directory. Jeśli tak — wyświetla ekran wypinania z domeny; jeśli jest już w `WORKGROUP` — od razu otwiera kreator migracji profilu.
- **Lokalne wypięcie z domeny (bez kontaktu z kontrolerem AD):** Wykorzystuje `NetUnjoinDomain` w trybie lokalnym, przypisuje maszynę do grupy `WORKGROUP` (`NetJoinDomain`) i czyści sufiksy DNS (`Domain`, `NV Domain`) w `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters`.
- **Kontrola aktywnych sesji użytkowników (WTS API):** Przed wypięciem z domeny oraz przed migracją wykrywa inne zalogowane lub rozłączone sesje (`WTSEnumerateSessions`) i pozwala wylogować je jednym kliknięciem (`WTSLogoffSession`), aby zwolnić blokady pliku `NTUSER.DAT`.
- **Automatyczne wykrywanie konta Entra ID:** Uruchamia systemowy kreator `ms-device-enrollment:?mode=aadj`, monitoruje w tle klucz `HKLM\SYSTEM\CurrentControlSet\Control\CloudDomainJoin\JoinInfo` i automatycznie uzupełnia adres e-mail użytkownika, a następnie tłumaczy go na identyfikator SID (`S-1-12-1-...`).
- **Migracja In-Place (w miejscu):** Nowy profil Entra ID przejmuje dotychczasowy katalog `C:\Users\<użytkownik>` bez kopiowania gigabajtów danych.
- **Kompleksowa obróbka rejestru i powłoki:**
  - Rekurencyjna zmiana właściciela, uprawnień DACL i podmiana starego SID na nowy SID w `NTUSER.DAT` oraz `UsrClass.dat`.
  - Czyszczenie starych tożsamości WAM/TokenBroker/Workplace Join/OOBE.
  - Zachowanie skojarzeń przeglądarek i plików PDF poprzez tymczasową dezaktywację sterownika `UCPD` oraz zadanie Fazy II przy rozruchu.
  - Konfiguracja jednorazowego autostartu OneDrive (`RunOnce`) zapobiegająca miganiu ikon na pulpicie podczas pierwszego logowania.
  - Adaptacja zmapowanych dysków sieciowych (`HKCU\Network\*`) do pracy poza domeną (obsługa udziałów SMB i wielopoziomowego DFS).
  - Reset bufora biometrycznego Windows Hello (`WbioSrvc`) i wymuszenie logowania hasłem (`PasswordCredentialProvider`) przy pierwszym logowaniu po migracji.

---

## Wymagania systemowe

- **System operacyjny:** Windows 10 / Windows 11 (x64).
- **Środowisko uruchomieniowe:** .NET 10.0 Desktop Runtime (`net10.0-windows`, `win-x64`).
- **Uprawnienia:** Konto lokalnego administratora (aplikacja wymaga elewacji UAC `requireAdministrator` zdefiniowanej w `Assets\app.manifest` i wykonuje migrację z osobnego konta administracyjnego, podczas gdy migrowany profil jest wylogowany).

---

## Architektura i struktura projektu

```text
C:\Migrator
├── Assets\
│   ├── app.ico                     # Ikona aplikacji
│   └── app.manifest                # Manifest UAC (requireAdministrator)
├── Core\
│   ├── BootCleanupService.cs       # Faza II: czyszczenie UCPD i asocjacji przy rozruchu jako SYSTEM
│   ├── FileSystemService.cs        # Migracja ACL plików/katalogów, czyszczenie cache powłoki, synchronizacja DPAPI Protect
│   ├── MigratorCore.cs             # Główny orkiestrator procesu migracji (Faza I)
│   ├── RegistryService.cs          # Montowanie/odmontowywanie gałęzi, translacja SID, czyszczenie kluczy, naprawa dysków i OneDrive
│   └── SystemConfigService.cs      # Operacje domenowe, sesje WTS, tłumaczenie SID Entra ID, biometria WbioSrvc, LogonUI/Winlogon
├── Models\
│   └── UserProfile.cs              # Model profilu użytkownika (SID, ProfilePath, Username)
├── UI\Controls\
│   ├── FlatProgressBar.cs          # Niestandardowy pasek postępu
│   ├── WizardForm.cs               # Logika wieloetapowego kreatora GUI
│   └── WizardForm.Designer.cs      # Definicja układu interfejsu użytkownika (High-DPI PerMonitorV2)
├── Utilities\
│   ├── Logger.cs                   # Zapis logów do C:\ProgramData\EntraProfileMigrator\Migration.log
│   ├── NativeMethods.cs            # Deklaracje P/Invoke (advapi32, netapi32, wtsapi32)
│   └── PrivilegeEscalation.cs      # Aktywacja przywilejów tokenu procesu (SeTakeOwnership, SeRestore, SeBackup, SeSecurity)
├── Migrator.csproj                 # Plik projektu .NET 10 (PublishSingleFile, win-x64)
└── Program.cs                      # Punkt wejścia aplikacji (obsługa GUI oraz przełącznika /cleanup)
```

---

## Przebieg działania krok po kroku

### Etap 0: Lokalne wypięcie z domeny Active Directory
*(Wyświetlany tylko wtedy, gdy `SystemConfigService.IsPartOfDomain()` zwróci `true`)*

1. **Weryfikacja roli maszyny:** Sprawdza w `HKLM\SYSTEM\CurrentControlSet\Control\ProductOptions`, czy komputer nie jest kontrolerem domeny (`LanmanNT`).
2. **Weryfikacja zalogowanych sesji:** Sprawdza przez `WTSEnumerateSessions`, czy w tle nie są zalogowani inni użytkownicy. Jeśli tak, oferuje ich automatyczne wylogowanie (`WTSLogoffSession`).
3. **Wypięcie lokalne (`RemoveComputerFromDomainAsync`):**
   - Wywołuje `NetUnjoinDomain(null, null, null, 0)` — wypina maszynę lokalnie bez konieczności komunikacji z kontrolerem domeny (unika błędów `Access Denied` w sieciach z ograniczonym dostępem lub poza VPN).
   - Przypisuje komputer do grupy roboczej `WORKGROUP` (`NetJoinDomain`).
   - Czyści wpisy `Domain` oraz `NV Domain` w `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters`.
4. **Restart:** Po 10-sekundowym odliczaniu wykonuje restart systemu (`shutdown.exe /r /t 0`).

---

### Etap 1: Wybór profilu źródłowego
*(Wyświetlany po uruchomieniu aplikacji na maszynie w `WORKGROUP`)*

1. Aplikacja odczytuje listę profili z `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList`.
2. Automatycznie odfiltrowuje konta systemowe, aktualnie zalogowane konto administratora wykonującego migrację oraz konto techniczne `bgates`.
3. Domyślnie wyświetla profile lokalne/domenowe (oznaczone kolorem jasnoniebieskim); po zaznaczeniu opcji *Pokaż wszystkie profile* wyświetla również istniejące profile Entra ID (`S-1-12-1-...`, kolor miętowy).

---

### Etap 2: Dołączenie do Entra ID i wskazanie konta docelowego

1. **Podłączenie do Entra ID:** Kliknięcie przycisku **Podłącz komputer do Entra ID** otwiera systemowe okno `ms-device-enrollment:?mode=aadj`.
2. **Automatyczne wykrycie adresu e-mail:** Wątek w tle przez maksymalnie 5 minut monitoruje klucz `HKLM\SYSTEM\CurrentControlSet\Control\CloudDomainJoin\JoinInfo` i po zakończeniu rejestracji automatycznie wpisuje odczytany `UserEmail` w pole tekstowe.
3. **Rozwiązanie SID konta Entra ID (`GetEntraIdSidByEmail`):**
   - **Metoda podstawowa:** Wykorzystuje natywne tłumaczenie `new NTAccount(@"AzureAD\" + email).Translate(typeof(SecurityIdentifier))` (działa natychmiast po dołączeniu urządzenia do Entra ID, bez konieczności wcześniejszego logowania użytkownika).
   - **Metoda zapasowa:** Przeszukuje pamięć podręczną `HKLM\SOFTWARE\Microsoft\IdentityStore\Cache\<tenant>\Identities` pod kątem `IdentityName` / `UserName`.
4. **Opcje konta źródłowego:** Administrator może opcjonalnie zaznaczyć wyłączenie (`net user /active:no`) lub usunięcie (`net user /delete`) lokalnego konta źródłowego.
5. **Walidacja blokad:** Przed startem migracji aplikacja sprawdza, czy nie ma innych aktywnych sesji WTS oraz czy plik `NTUSER.DAT` profilu źródłowego nie jest zablokowany (`FileSystemService.IsFileLocked`).

---

### Etap 3: Migracja profilu In-Place (Faza I)

Główny proces realizowany przez `MigratorCore.RunMigrationAsync`:

1. **Eskalacja przywilejów (`PrivilegeEscalation.EnablePrivileges`):**
   Włącza w tokenie procesu uprawnienia `SeTakeOwnershipPrivilege`, `SeRestorePrivilege`, `SeBackupPrivilege` oraz `SeSecurityPrivilege`.
2. **Dezaktywacja sterownika UCPD (`SystemConfigService.DisableUcpdService`):**
   Ustawia `Start = 4` (Disabled) dla usługi `HKLM\SYSTEM\CurrentControlSet\Services\UCPD` i wyłącza zadanie `\Microsoft\Windows\AppxDeploymentClient\UCPD velocity`, aby umożliwić modyfikację chronionych kluczy asocjacji (`http`, `https`, `.pdf`).
3. **Migracja uprawnień systemu plików (`FileSystemService.MigrateDirectoryPermissions`):**
   Przechodzi iteracyjnie przez cały katalog profilu (z pominięciem linków symbolicznych `ReparsePoint`), ustawia nowego właściciela (`targetSid`), nadaje pełną kontrolę (`FullControl`) z dziedziczeniem i usuwa wpisy ACL należące do starego SID (`sourceSid`).
4. **Przetwarzanie gałęzi rejestru (`RegistryService.ProcessRegistryHive`):**
   Montuje kolejno `NTUSER.DAT` (jako `HKLM\TempHive_Migrate_NTUSER`) oraz `AppData\Local\Microsoft\Windows\UsrClass.dat` (jako `HKLM\TempHive_Migrate_UsrClass`) za pomocą `RegLoadKey`:
   - **Translacja ACL i SID (`TranslateRegistryKey`):** Rekurencyjnie nadaje nowemu SID uprawnienia `FullControl` i właściciela na wszystkich podkluczach oraz podmienia wystąpienia tekstowe starego SID na nowy SID w wartościach `REG_SZ` i `REG_EXPAND_SZ`.
   - **Czyszczenie kluczy tożsamości i powłoki (w `NTUSER.DAT`):** Usuwa (z wymuszeniem przejęcia własności i usunięciem reguł `Deny`) klucze z tablic `identityKeys` oraz `keysToDelete` (m.in. `IdentityStore`, `TokenBroker`, `WebAccountManager`, `Workplace Join`, `AuthBroker`, `UserOOBE`, `CloudExperienceHost`, `CloudStore`, `Active Setup\Installed Components`, `OOBE`, `UserProfileEngagement` oraz skojarzenia `http`/`https`/`pdf`).
   - **Autostart OneDrive (`EnsureOneDriveFirstLogonStart`):** Kopiuje komendę startową OneDrive z `HKCU\...\Run` do `HKCU\...\RunOnce` (`OneDriveFirstLogon`), aby przy pierwszym logowaniu po migracji OneDrive wystartował natychmiast, zapobiegając miganiu ikon na pulpicie.
   - **Adaptacja zmapowanych dysków sieciowych (`FixMappedDrives`):** Aktualizuje wszystkie podklucze w `HKCU\Network\*`, ustawiając:
     - `UserName = ""` (`REG_SZ`) — wartość wymagana przez `MprGetKeyInfo` w `mpr.dll`, aby połączenie korzystało z bieżących poświadczeń sesji/Menedżera poświadczeń zamiast starego konta domenowego,
     - `ProviderFlags = 1` (`REG_DWORD`) — flaga informująca `ntlanman.dll` o ścieżce DFS,
     - `DeferFlags = 4` (`REG_DWORD`) — `CONNECT_DEFERRED`, dzięki czemu dyski sieciowe są widoczne od razu po zalogowaniu i nie są odrzucane przed uwierzytelnieniem,
     - `ConnectionType = 1`, `ConnectFlags = 0` oraz usunięcie `UseOptions`.
   - Wymusza odśmiecanie pamięci (`GC.Collect()`) i odmontowuje gałęzie przez `RegUnLoadKey`.
5. **Aktualizacja `ProfileList` (`RegistryService.UpdateProfileList`):**
   Tworzy kopię zapasową starego klucza profilu jako `<source.SID>.bak`, usuwa oryginalny klucz `<source.SID>` i tworzy nowy klucz `<target.SID>` wskazujący na dotychczasowy `ProfileImagePath` ze stanem `State = 0`.
6. **Czyszczenie pamięci podręcznych powłoki i tokenów (`FileSystemService.CleanupShellCache`):**
   Zmienia nazwy 12 folderów cache (m.in. `WebCache`, `Explorer`, `IdentityCache`, `TokenBroker`, `Microsoft.AAD.BrokerPlugin_cw5n1h2txyewy`, `Credentials`, `Vault`, `Teams`) na `*_MigratedBak`, wymuszając ich czystą odbudowę przy logowaniu na konto Entra ID.
7. **Synchronizacja kluczy DPAPI (`FileSystemService.SyncTargetProtectKeys`):**
   Jeśli konto docelowe miało wcześniej osobny folder profilu, synchronizuje klucze z `AppData\Roaming\Microsoft\Protect\<targetSid>` do migrowanego profilu.
8. **Reset danych biometrycznych (`SystemConfigService.ResetBiometricData`):**
   Zatrzymuje usługę `WbioSrvc`, usuwa referencje do starego SID z `WinBio\AccountInfo` i `WbioSrvc\Parameters\Databases`, czyści pliki `*.DAT` z `C:\Windows\System32\WinBioDatabase` i ponownie uruchamia usługę, aby uniknąć konfliktu Windows Hello ("To rozpoznawanie twarzy/PIN jest już używane na innym koncie").
9. **Konfiguracja ekranu logowania (`SystemConfigService.SetDefaultLogonUser`):**
   Ustawia nowego użytkownika jako domyślnego w `Winlogon` oraz `LogonUI` i wymusza dostawcę hasła (`PasswordCredentialProvider`: `{60b78e88-ead8-445c-9cfd-0b87f74ea6cd}`) na kafelku użytkownika (`LogonUI\UserTile`), aby pierwsze logowanie po migracji odbyło się hasłem (co pozwala poprawnie zainicjować sesję LSA, dostęp do udziałów SMB/DFS i kreator nowego PIN-u Windows Hello).
10. **Rejestracja zadania Fazy II (`SystemConfigService.ScheduleCleanupTask`) i sprzątanie `C:\NTUSER.DAT`:**
    Rejestruje zadanie `EntraMigratorCleanup` uruchamiane przy starcie systemu jako `NT AUTHORITY\SYSTEM`, usuwa ewentualne osierocone pliki `C:\NTUSER.DAT*` i inicjuje restart komputera.

---

### Etap 4: Czyszczenie po restarcie (Faza II)

Przy starcie systemu (zanim jakikolwiek użytkownik się zaloguje i zanim sterownik `UCPD` zablokuje klucze rejestru) Harmonogram Zadań uruchamia proces `Migrator.exe /cleanup "<ścieżka_profilu>"` (`BootCleanupService.RunPostBootCleanup`):

1. Montuje `NTUSER.DAT` w trybie offline (`HKLM\TempHive_BootCleanup`).
2. Usuwa klucze asocjacji (`http`, `https`, `pdf`, `.pdf`), jeśli w Fazie I były zablokowane przez aktywny w pamięci sterownik `UCPD`.
3. Odmontowuje rejestr, przywraca domyślny stan sterownika `UCPD` (`Start = 1`) oraz zadania `UCPD velocity`, a następnie usuwa zadanie `EntraMigratorCleanup` z harmonogramu zadań.

---

## Szczegóły techniczne kluczowych mechanizmów

| Obszar | Rozwiązanie zastosowane w aplikacji | Cel / Efekt |
| :--- | :--- | :--- |
| **Udziały sieciowe SMB i DFS (`HKCU\Network`)** | `UserName = ""`, `ProviderFlags = 1`, `DeferFlags = 4` | Biblioteka `mpr.dll` (`MprGetKeyInfo`) wymaga istnienia pustej wartości tekstowej `UserName=""`, aby nie odrzucić zmapowanego dysku podczas logowania. `DeferFlags = 4` zapobiega znikaniu dysku poza domeną i pozwala na dostęp bez podawania poświadczeń po zalogowaniu hasłem oraz po zapisaniu poświadczeń w Menedżerze poświadczeń przy logowaniu PIN-em. |
| **Pierwszy start OneDrive i ikony pulpitu** | Kopiowanie wpisu OneDrive z `Run` do `RunOnce` (`OneDriveFirstLogon`) | Po usunięciu `Active Setup\Installed Components` Windows przebudowuje środowisko użytkownika przy pierwszym logowaniu i opóźnia wykonanie zwykłego klucza `Run`. Wpis w `RunOnce` uruchamia OneDrive od razu i eliminuje miganie ikon synchronizowanych z OneDrive KFM. |
| **Pierwsze logowanie po migracji** | Wymuszenie GUID `{60b78e88-ead8-445c-9cfd-0b87f74ea6cd}` w `LogonUI` | Gwarantuje, że użytkownik zaloguje się po raz pierwszy hasłem Entra ID (a nie starym lub nieaktywnym kafelkiem PIN), co poprawnie buduje sesję Kerberos/NTLM i uruchamia kreator nowego PIN-u Windows Hello. |

---

## Logowanie i diagnostyka

Wszystkie operacje (zarówno z etapu wypinania z domeny, Fazy I migracji, jak i Fazy II po restarcie) są szczegółowo zapisywane w czasie rzeczywistym w dwóch miejscach:

1. **Konsola w oknie aplikacji (Strona 3 kreatora):** Kolorowane komunikaty statusu (`BŁĄD` – czerwony, `OSTRZEŻENIE` – złoty, nagłówki `===` – cyjan, podsumowania – zielony).
2. **Plik logu na dysku:**
   ```text
   C:\ProgramData\EntraProfileMigrator\Migration.log
   ```
   Zawiera pełną historię ze znacznikami czasu (`[yyyy-MM-dd HH:mm:ss]`), statystykami przetworzonych katalogów/plików, liczbą przetłumaczonych kluczy i wartości rejestru, listą usuniętych kluczy i zarchiwizowanych folderów cache oraz pełnymi śladami stosu (`StackTrace`) w przypadku wyjątków.

---

## Kompilacja i publikacja

Projekt jest skonfigurowany do kompilacji jako pojedynczy plik wykonywalny (`PublishSingleFile = true`, `RuntimeIdentifier = win-x64`, `SelfContained = false`).

### Kompilacja deweloperska (Debug):
```powershell
dotnet build c:\Migrator\Migrator.csproj
```

### Publikacja do pojedynczego pliku `.exe` (Release):
```powershell
dotnet publish c:\Migrator\Migrator.csproj -c Release -r win-x64
```
Wynikowy plik wykonywalny zostanie wygenerowany w katalogu:
`c:\Migrator\bin\Release\net10.0-windows\win-x64\publish\Migrator.exe`

