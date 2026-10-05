# PLSQL Retirement Windows Cursor Developer Setup

Developer procedure for setting up an Amazon WorkSpace to build and run the CustomerService API. This guide records the sequence verified on a Windows WorkSpace in September 2026\. Use your own GitLab identity and personal token. Never commit a token or copy the package publishing bot token to another developer.

# **What is required**

Install Git for Windows, Cursor, the .NET 8 SDK for the net8.0 projects, and the .NET 10 SDK required by the tested DotRush extension. Cursor uses DotRush and .NET Install Tool for C\#; Microsoft C\# Dev Kit is not available in Cursor. The private OracleDAL package is referenced by CustomerService only, not the ExceptionHandler project.

# **1 Install Git Bash and configure Git**

The tested Amazon WorkSpace did not have winget. Download the 64 bit Git for Windows installer from https://git-scm.com/download/win. Keep the standard options, including bundled OpenSSH and Git Credential Manager. Start a new Git Bash window and verify:

git \--version

Verified example: git version 2.55.0.windows.5. Set your own GitLab email and verify the credential helper:

git config \--global user.name "YOUR NAME"  
git config \--global user.email "YOUR\_GITLAB\_EMAIL"  
git config \--show-origin \--get credential.helper

If the helper check returns nothing, run git config \--global credential.helper manager and verify it returns manager. A successful git config write usually prints nothing.

# **2 Clone and open the repository**

In Git Bash, use the GitLab HTTPS clone URL. Git Credential Manager may open browser authentication. Open the cloned folder through Cursor File \> Open Folder; the Cursor command line launcher is optional.

mkdir \-p \~/66DegreesRepos/repos  
cd \~/66DegreesRepos/repos  
git clone https://gitlab.com/ftdr/data-engineering/plsql-retirement.git  
cd plsql-retirement  
git status

Use the actual path created on your WorkSpace if your home directory is on a drive other than C:. The repository was verified with CustomerService/CustomerService.csproj and PLSQL-Retirement.ExceptionHandler/PLSQL-Retirement.ExceptionHandler.csproj, both targeting net8.0. Check the target frameworks in the current checkout before choosing an SDK.

grep \-n "TargetFramework" CustomerService/CustomerService.csproj PLSQL-Retirement.ExceptionHandler/PLSQL-Retirement.ExceptionHandler.csproj

# **3 Install the SDKs and Cursor extensions**

Install the Windows x64 .NET 8 SDK from https://dotnet.microsoft.com/download/dotnet/8.0. The tested version was 8.0.425. Install a .NET 10 SDK alongside it because the tested DotRush release reported that its Roslyn server requires SDK 10 or newer. Installing SDK 10 does not change the applications’ net8.0 target. Restart Cursor and Git Bash after installation.

dotnet \--list-sdks  
dotnet \--info

In Cursor, open Extensions with Ctrl+Shift+X. Following https://cursor.com/for/csharp, install DotRush and .NET Install Tool. Do not install Microsoft C\# Dev Kit. Open a .cs file, allow DotRush to initialize, and confirm hover information, Go to Definition, and IntelliSense. If DotRush prompts for a project, run DotRush: Pick Project or Solution files and select the two project files. If it cannot find an SDK, verify .NET 10 is listed before changing any Cursor settings. DotRush supports C\# files; it does not provide Razor or Blazor specific editing support.

# **4 Verify NuGet sources**

dotnet nuget list source \--format detailed

NuGet.org must point to https://api.nuget.org/v3/index.json. The repository originally used https://nuget.org, which caused NU1301 because that page is not a NuGet v3 JSON index. The configured private feed is https://gitlab.com/api/v4/projects/85992194/packages/nuget/index.json. If the public URL is wrong, correct it in the repository configuration through the architect’s approved change process; a local update can leave nuget.config modified. Do not commit workstation credentials.

# **5 Create a personal token and authenticate**

In your personal GitLab profile, open Access tokens \> Generate Token \> Legacy Token. Create a token with the api scope and an expiration date allowed by company policy. Your account must have access to GitLab project 85992194\. Save the token in the approved password manager. Do not create it under the project’s access-token settings and do not reuse the nuget-upload bot token.

In Git Bash, use the username returned by the GitLab API, without the @ shown beside a handle. The validated username on the architect’s WorkSpace was alexandre.little; every other developer must substitute their own. Paste into the hidden prompt using right-click or Shift+Insert if Ctrl+V does not work.

export GITLAB\_USERNAME="YOUR\_GITLAB\_USERNAME\_WITHOUT\_AT"  
read \-rsp "Personal GitLab token: " GITLAB\_TOKEN  
echo  
export GITLAB\_TOKEN  
printf "Token length: %s\\n" "${\#GITLAB\_TOKEN}"

The length must be substantially greater than one. This API check identifies the owner of the token; verify that its username matches GITLAB\_USERNAME. Do not share the token or unredacted command output containing secrets.

curl \--silent \--show-error \--header "PRIVATE-TOKEN: $GITLAB\_TOKEN" https://gitlab.com/api/v4/user | grep \-o '"username":"\[^"\]\*"'

The repository currently includes a GitLab credential entry using Username=Private-Token and a %GITLAB\_TOKEN% variable reference. To make NuGet use each developer’s personal username and token for this Git Bash session, set the following higher-priority source credential variable. Never echo its value or put it in a repository file.

export NuGetPackageSourceCredentials\_GitLab="Username=${GITLAB\_USERNAME};Password=${GITLAB\_TOKEN};ValidAuthenticationTypes=Basic"

These exported variables last only for the current shell session. Re-enter the token and export the NuGet source credential variable in each new Git Bash session before an uncached private-package restore. A future repository-wide authentication change may replace this interim procedure.

# **6 Restore and verify the private package**

From the repository root, run the restores and inspect the direct package list. CustomerService must reference 66degrees.Data.OracleDAL version 1.0.0. The ExceptionHandler project does not require this package.

dotnet restore CustomerService/CustomerService.csproj \--no-http-cache \--verbosity minimal  
dotnet restore PLSQL-Retirement.ExceptionHandler/PLSQL-Retirement.ExceptionHandler.csproj \--verbosity minimal  
dotnet list CustomerService/CustomerService.csproj package \--no-restore

A 401 at the GitLab package URL usually means the token and Basic-auth username do not match. The @ prefix and literal placeholder YOUR\_GITLAB\_USERNAME caused 401s during verification. If authentication succeeds but the package is absent, confirm the latest CustomerService.csproj change was committed, pushed, and pulled.

# **7 Create the local Development settings file**

Environment-specific application settings are intentionally excluded from source control. Each developer must create a local appsettings.Development.json file in the root of every API project they need to run, beside Program.cs and the project .csproj file. Do not place this file under Api, Infrastructure, Properties, or Tests.

In Cursor Explorer, right-click the API project folder, select New File, and name the file appsettings.Development.json. For CustomerService, the path is CustomerService/appsettings.Development.json. For ContractService, the path is ContractService/appsettings.Development.json.

Use this structure and replace the placeholders with the architect-approved local Oracle values:

{  
&nbsp;&nbsp;"Logging": {  
&nbsp;&nbsp;&nbsp;&nbsp;"LogLevel": {  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"Default": "Information",  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"Microsoft.AspNetCore": "Warning"  
&nbsp;&nbsp;&nbsp;&nbsp;}  
&nbsp;&nbsp;},  
&nbsp;&nbsp;"ConnectionStrings": {  
&nbsp;&nbsp;&nbsp;&nbsp;"Oracle": "User Id=YOUR\_DB\_USERNAME;Password=YOUR\_DB\_PASSWORD;Data Source=YOUR\_TNS\_ALIAS;Tns\_Admin=D:\\\\Oracle\\\\Wallets\\\\YOUR\_WALLET\_FOLDER;Wallet\_Location=D:\\\\Oracle\\\\Wallets\\\\YOUR\_WALLET\_FOLDER;"  
&nbsp;&nbsp;}  
}

The wallet path must point to the extracted wallet directory containing cwallet.sso, sqlnet.ora, and tnsnames.ora. Do not point the .NET connection string to the ZIP file. Do not copy wallet files, database credentials, or populated environment settings into the repository. Connection strings may be created or changed only by an architect after coordination with the other architects.

Confirm that Git ignores the local file before building or committing:

git check-ignore \-v CustomerService/appsettings.Development.json  
git status \--short

The first command must identify a matching rule in the repository-root .gitignore. The settings file must not appear as added or modified in git status. If it does, stop and contact an architect before committing. ASP.NET Core loads appsettings.Development.json when the application starts with the Development environment.

# **8 Build and start the API**

dotnet build CustomerService/CustomerService.csproj \--no-restore  
dotnet build PLSQL-Retirement.ExceptionHandler/PLSQL-Retirement.ExceptionHandler.csproj \--no-restore  
dotnet run \--project CustomerService/CustomerService.csproj \--no-restore

The verified API started in Development on http://localhost:5171. While it is still running, open a second Git Bash window and check the Swagger page. Curl may run from any directory. An HTTP 000 usually means the API is no longer running or the connection failed; use \--show-error to see why.

curl \--silent \--show-error \--max-time 5 \--output /dev/null \--write-out "Swagger HTTP status: %{http\_code}\\n" http://127.0.0.1:5171/swagger/index.html

Verified result: HTTP 200\. Stop the running API with Ctrl+C. Actual ports can vary with the project launch settings.

# **Verified results and current limitations**

* Git Bash, Cursor, DotRush language features, and both SDKs worked on the tested WorkSpace.  
* Both projects restored and built; OracleDAL 1.0.0 appeared as a direct CustomerService package. Swagger returned 200\.  
* Two build warnings remain: CS7022 for xUnit’s generated entry point alongside API top-level code, and CS8618 for the uninitialized RepositoryWrapper.\_customer field.  
* CustomerService/Tests/CustomerServiceTests.cs exists, but xUnit v3 is referenced in the API project. dotnet test under SDK 10 failed because the current Microsoft.Testing.Platform configuration uses the unsupported VSTest mode. Tests are not verified; test structure and runner configuration are a separate codebase decision.

# **Repository hygiene follow up for the architect**

Do not treat the following as a developer installation step. After this guide is delivered, review the repository because bin and obj outputs are tracked and a normal restore or build marks many generated files modified or untracked. Add appropriate ignore rules and stop tracking generated outputs through a reviewed repository change. Review the committed nuget.config: its %GITLAB\_TOKEN% value is a variable reference, not a committed secret, but its public feed URL and personal-token username strategy need standardization. Check the local nuget.config diff for actual credentials before any commit. Avoid git add . until this work is complete. The test-project layout and two build warnings are separate backlog items.

Before pushing application changes, inspect git status and git diff \--cached. Stage only intended source files. Do not commit NuGet tokens, workstation configuration changes, bin or obj artifacts, or downloaded .nupkg files.