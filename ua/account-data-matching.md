<table>
    <thead>
        <tr>
            <th>Заголовок</th>
            <th>Цілі</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td rowspan=3 align="center">Відповідність даних аккаунту</td>
            <td align="center">Пояснити ризики безпеки, пов'язані з відсутністю перевірок валідації даних</td>
        </tr>
        <tr>
            <td align="center">Реалізувати перевірки валідації даних, використовуючи довгий синтаксис Rust</td>
        </tr>
        <tr>
            <td align="center">Реалізувати перевірки валідації даних, використовуючи обмеження Anchor</td>
        </tr>
    </tbody>
</table>

# Коротко кажучи (TL;DR):
Використовуйте перевірки валідації даних, щоб перевірити, чи відповідають дані облікового запису очікуваному значенню. Без належних перевірок валідації даних можуть використовуватися неочікувані облікові записи в інструкції.

Для реалізації перевірок валідації даних у Rust просто порівнюйте дані, збережені на обліковому записі, з очікуваним значенням.

```rust
if ctx.accounts.user.key() != ctx.accounts.user_data.user {
    return Err(ProgramError::InvalidAccountData.into());
}
```

У Anchor ви можете використовувати `constraint`, щоб перевірити, чи вираз відповідає true. Альтернативно ви можете використовувати `has_one`, щоб перевірити, що поле цільового облікового запису, збережене на обліковому записі, відповідає ключу облікового запису в структурі Accounts.

## Огляд:

Відповідність даних облікового запису передбачає перевірку валідації даних для перевірки того, чи дані, збережені на обліковому записі, відповідають очікуваному значенню. Перевірки валідації даних надають можливість включити додаткові обмеження для забезпечення того, що в інструкцію передаються відповідні облікові записи.

Це може бути корисно, коли облікові записи, які необхідні для виконання інструкції, мають залежності від значень, збережених в інших облікових записах, або якщо інструкція залежить від даних, збережених на обліковому записі.

## Відсутність перевірки валідації даних:

У прикладі нижче є інструкція `update_admin`, яка оновлює поле admin, збережене на обліковому записі admin_config.

Інструкція не має перевірки валідації даних для перевірки того, чи адміністратор облікового запису, який підписує транзакцію, відповідає адміністратору, збереженому на обліковому записі admin_config. Це означає, що будь-який обліковий запис, який підписує транзакцію і передається в інструкцію як обліковий запис адміністратора, може оновлювати обліковий запис admin_config.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

## Додавання перевірки валідації даних:

Базовий підхід у Rust для вирішення цієї проблеми полягає в простому порівнянні переданого ключа адміністратора з ключем адміністратора, збереженим на обліковому записі admin_config, і генерації помилки, якщо вони не відповідають одне одному.

```rust
if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
    return Err(ProgramError::InvalidAccountData.into());
}
```

Додавши перевірку валідації даних, інструкція update_admin буде виконуватися лише у випадку, якщо підписник транзакції адміністратора відповідає адміністратору, збереженому на обліковому записі admin_config.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
      if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
            return Err(ProgramError::InvalidAccountData.into());
        }
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

## Використання обмежень Anchor:

Anchor спрощує цей процес за допомогою обмеження has_one. Ви можете використовувати обмеження has_one для переміщення перевірки валідації даних з логіки інструкції до структури UpdateAdmin.

У прикладі нижче has_one = admin вказує, що адміністратор облікового запису, який підписує транзакцію, повинен відповідати полю admin, збереженому на обліковому записі admin_config. Для використання обмеження has_one конвенція найменування поля даних на обліковому записі повинна відповідати найменуванню структури валідації облікового запису.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        has_one = admin
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

Альтернативно ви можете використовувати обмеження constraint для вручну додавання виразу, який повинен обчислитися в true, щоб виконання продовжувалося. Це корисно, коли з якої-небудь причини неможливо дотриматися конвенції найменування або коли потрібен більш складний вираз для повної перевірки вхідних даних.

```rust
#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        constraint = admin_config.admin == admin.key()
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}
```

# Демонстрація:

Для цієї демонстрації ми створимо просту програму "сховище" (vault), схожу на програму, яку ми використовували в уроках з авторизації підписника та перевірки власника. Як і в тих демонстраціях, ми покажемо, як відсутність перевірки валідації даних може дозволити розсмоктати сховище.

## 1. Початковий код:

Для початку завантажте стартовий код з гілки "starter" цього репозиторію. Стартовий код включає програму з двома інструкціями та заготовку для тестового файлу.

Інструкція initialize_vault ініціалізує новий обліковий запис Vault та новий TokenAccount. Обліковий запис Vault буде зберігати адресу облікового запису токенів, авторитет сховища та обліковий запис токенів для виведення.

Авторитет нового облікового запису токенів буде встановлено як обліковий запис сховища, PDA програми. Це дозволяє обліковому запису сховища підписувати переміщення токенів з облікового запису токенів.

Інструкція insecure_withdraw переказує всі токени на обліковому записі токенів сховища до облікового запису токенів для виведення.

Зверніть увагу, що ця інструкція має перевірку підписника для авторитету та перевірку власника для сховища. Однак ніде в логіці перевірки облікового запису чи логіці інструкції немає коду, який перевіряє, що обліковий запис авторитету, переданий в інструкцію, відповідає обліковому запису авторитету на сховищі.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
        ctx.accounts.vault.withdraw_destination = ctx.accounts.withdraw_destination.key();
        Ok(())
    }

    pub fn insecure_withdraw(ctx: Context<InsecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeVault<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32 + 32,
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        init,
        payer = authority,
        token::mint = mint,
        token::authority = vault,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct InsecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
    withdraw_destination: Pubkey,
}
```

## 2. Тест insecure_withdraw інструкції:

Щоб довести, що це проблема, напишемо тест, в якому рахунок, інший, ніж авторитет сховища, намагається вивести кошти з сховища.

Тестовий файл включає код для виклику інструкції initialize_vault, використовуючи гаманець постачальника як авторитет, а потім виводить 100 токенів на обліковий запис токенів сховища.

Додайте тест для виклику інструкції insecure_withdraw. Використовуйте withdrawDestinationFake як обліковий запис виводу та walletFake як авторитет. П

отім відправте транзакцію за допомогою walletFake.

Оскільки немає жодних перевірок, які перевіряли б, чи обліковий запис авторитету, переданий в інструкцію, відповідає значенням, збереженим на сховищі, інструкція буде оброблена успішно, і токени будуть переведені на обліковий запис виводу withdrawDestinationFake.

```javascript
describe("account-data-matching", () => {
  ...
  it("Insecure withdraw", async () => {
    const tx = await program.methods
      .insecureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestinationFake,
        authority: walletFake.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Запустіть anchor test, щоб переконатися, що обидві транзакції завершаться успішно.

```
account-data-matching
  ✔ Initialize Vault (811ms)
  ✔ Insecure withdraw (403ms)
```

## 3. Додавання інструкції secure_withdraw:

Давайте реалізуємо безпечну версію цієї інструкції, яку ми назвемо secure_withdraw.

Ця інструкція буде ідентичною інструкції insecure_withdraw, за винятком того, що ми використовуватимемо обмеження has_one в структурі валідації облікового запису (SecureWithdraw), щоб перевірити, що обліковий запис авторитету, переданий в інструкцію, відповідає обліковому запису авторитету на сховищі. Таким чином, лише правильний обліковий запис авторитету може вивести токени сховища.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;
    ...
    pub fn secure_withdraw(ctx: Context<SecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
        has_one = token_account,
        has_one = authority,
        has_one = withdraw_destination,

    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}
```

## 4. Тест інструкції secure_withdraw:

Тепер давайте протестуємо інструкцію secure_withdraw двома тестами: один, який використовує walletFake як авторитет, і один, який використовує wallet як авторитет. Ми очікуємо, що перший виклик поверне помилку, а другий пройде успішно.

```javascript
describe("account-data-matching", () => {
  ...
  it("Secure withdraw, expect error", async () => {
    try {
      const tx = await program.methods
        .secureWithdraw()
        .accounts({
          vault: vaultPDA,
          tokenAccount: tokenPDA,
          withdrawDestination: withdrawDestinationFake,
          authority: walletFake.publicKey,
        })
        .transaction()

      await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })

  it("Secure withdraw", async () => {
    await spl.mintTo(
      connection,
      wallet.payer,
      mint,
      tokenPDA,
      wallet.payer,
      100
    )

    await program.methods
      .secureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestination,
        authority: wallet.publicKey,
      })
      .rpc()

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Запустіть `anchor test`, щоб переконатися, що транзакція, використовуючи неправильний обліковий запис авторитету, тепер поверне помилку Anchor, тоді як транзакція з використанням правильних облікових записів завершиться успішно.

```
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS invoke [1]',
'Program log: Instruction: SecureWithdraw',
'Program log: AnchorError caused by account: vault. Error Code: ConstraintHasOne. Error Number: 2001. Error Message: A has one constraint was violated.',
'Program log: Left:',
'Program log: DfLZV18rD7wCQwjYvhTFwuvLh49WSbXFeJFPQb5czifH',
'Program log: Right:',
'Program log: 5ovvmG5ntwUC7uhNWfirjBHbZD96fwuXDMGXiyMwPg87',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS consumed 10401 of 200000 compute units',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS failed: custom program error: 0x7
```

d1'
Зверніть увагу, що в журналах Anchor вказано обліковий запис, який спричинив помилку (AnchorError caused by account: vault).

✔ Secure withdraw, expect error (77ms)
✔ Secure withdraw (10073ms)
І от ви вже закрили уразливість щодо безпеки. Тема більшості таких можливих використань полягає в тому, що вони досить прості. Однак, з ростом обсягу та складності ваших програм, стає все легше пропустити можливі використання. Це чудово вводити звичку писати тести, які надсилають інструкції, які не повинні працювати. Чим їх більше, тим краще. Таким чином ви виявляєте проблеми до введення в експлуатацію.

Якщо ви хочете переглянути код остаточного рішення, ви можете знайти його на гілці "solution" репозиторію.

# Завдання:
Так само, як і з іншими уроками цього модулю, ваша можливість практикувати уникнення цього експлойта безпеки полягає в перевірці ваших власних чи інших програм.

Відведіть час для перегляду принаймні однієї програми та переконайтеся, що належні перевірки даних встановлені для уникнення можливих експлойтів.

Пам'ятайте, якщо ви знайшли помилку чи експлойт в програмі когось іншого, будь ласка, повідомте його! Якщо ви знаходите їх у власній програмі, будьте впевнені, що виправте їх одразу.
