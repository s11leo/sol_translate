## TL;DR
Используйте проверки валидации данных для проверки того, что данные учетной записи соответствуют ожидаемому значению. Без соответствующих проверок валидации данных неожиданные учетные записи могут использоваться в инструкции.

Для реализации проверок валидации данных на Rust просто сравните данные, сохраненные на учетной записи, с ожидаемым значением.

```rust
if ctx.accounts.user.key() != ctx.accounts.user_data.user {
    return Err(ProgramError::InvalidAccountData.into());
}
```

В Anchor вы можете использовать `constraint` для проверки, вычисляется ли данное выражение как true. Также можно использовать `has_one` для проверки того, что поле целевой учетной записи, сохраненной в учетной записи, соответствует ключу учетной записи в структуре Accounts.

## Обзор
Соответствие данных учетной записи означает проверку данных, сохраненных на учетной записи, с ожидаемым значением. Проверки валидации данных предоставляют возможность включить дополнительные ограничения, чтобы гарантировать, что в инструкцию передаются соответствующие учетные записи.

Это может быть полезно, когда учетные записи, необходимые для выполнения инструкции, зависят от значений, хранящихся в других учетных записях, или если инструкция зависит от данных, хранящихся в учетной записи.

## Отсутствие проверки валидации данных
Приведенный ниже пример включает инструкцию `update_admin`, которая обновляет поле `admin`, сохраненное на учетной записи `admin_config`.

В инструкции отсутствует проверка валидации данных для подтверждения того, что учетная запись `admin`, подписывающая транзакцию, соответствует `admin`, сохраненному на учетной записи `admin_config`. Это означает, что любая учетная запись, подписывающая транзакцию и передаваемая в инструкцию в качестве учетной записи `admin`, может обновить учетную запись `admin_config`.

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

## Добавление проверки валидации данных
Базовый подход на Rust к решению этой проблемы - просто сравнить переданный ключ `admin` с ключом `admin`, сохраненным на учетной записи `admin_config`, и вызвать ошибку, если они не совпадают.

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

## Использование Anchor constraints
Anchor упрощает это с помощью ограничения `has_one`. Вы можете использовать `has_one` для перемещения проверки валидации данных из логики инструкции в структуру `UpdateAdmin`.

В приведенном ниже примере `has_one = admin` указывает, что учетная запись `admin`, подписывающая транзакцию, должна соответствовать полю `admin`, сохраненному на учетной записи `admin_config`. Для использования ограничения `has_one` согласованность именования поля данных на учетной записи должна быть согласованной с именованием структуры проверки учетной записи.

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

В качестве альтернативы можно использовать ограничение `constraint` для ручного добавления выражения, которое должно оцениваться как true, чтобы выполнение продолжалось. Это полезно, когда по какой-то причине именование не может быть соглас

ованным или когда вам нужно более сложное выражение для полной валидации входящих данных.

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
Извините за путаницу. Вот перевод вашего текста на русский язык:

```markdown

## Демо

Для этого демо мы создадим простую программу "хранилище" (vault), подобную программе, использованной в уроке по авторизации подписанта (Signer Authorization) и уроке проверки владельца (Owner Check). Как и в тех демонстрациях, мы покажем в этом демо, как отсутствие проверки валидации данных может позволить вытеканию из хранилища.

## 1. Начало

Для начала загрузите стартовый код с ветки `starter` из этого репозитория. В стартовом коде уже есть программа с двумя инструкциями и структура для тестового файла.

Инструкция `initialize_vault` инициализирует новую учетную запись Vault и новую учетную запись TokenAccount. Учетная запись Vault будет содержать адрес учетной записи токена, авторитет хранилища и учетную запись токена для вывода.

Авторитет новой учетной записи токена будет установлен как хранилище, PDA программы. Это позволяет учетной записи Vault подписывать трансфер токенов с учетной записи токена.

Инструкция `insecure_withdraw` переводит все токены в учетной записи токена хранилища на учетную запись токена `withdraw_destination`.

Обратите внимание, что у этой инструкции есть проверка подписанта на авторитет и проверка владельца для Vault. Однако нигде в проверке учетных записей или логике инструкции нет кода, который проверяет, что учетная запись авторитета, переданная в инструкцию, соответствует учетной записи авторитета в Vault.

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
    // ... (неизмененный)
}

#[derive(Accounts)]
pub struct InsecureWithdraw<'info> {
    // ... (неизмененный)
}

#[account]
pub struct Vault {
    // ... (неизмененный)
}
```

## 2. Тестирование инструкции insecure_withdraw

Чтобы продемонстрировать, что это проблема, давайте напишем тест, в котором учетная запись, отличная от авторитета хранилища, пытается вытянуть токены из хранилища.

Файл теста включает код для вызова инструкции `initialize_vault`, используя кошелек-поставщик в качестве авторитета, и затем эмитирует 100 токенов на учетную запись токена хранилища.

Добавьте тест для вызова инструкции `insecure_withdraw`. Используйте `withdrawDestinationFake` в качестве учетной записи `withdrawDestination` и `walletFake` в качестве авторитета. Затем отправьте транзакцию, используя `walletFake`.

Поскольку нет проверок, которые проверяют, что учетная запись авторитета, переданная в инструкцию, соответствует значениям, сохраненным в учетной записи Vault, инициализированной в первом тесте, инструкция будет успешно обработана, и токены будут переданы на учетную запись `withdrawDestinationFake`.

```javascript
describe("account-data-matching", () => {
  // ... (неизмененный)
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

Запустите `anchor test`, чтобы убедиться, что обе транзакции завершатся успешно.

```bash
account-data-matching
  ✔ Initialize Vault (811ms)
  ✔ Insecure withdraw (403ms)
```

## 3. Добавление инструкции secure_withdraw

Теперь давайте реализуем безопасную версию этой инструкции под названием `secure_withdraw`.

Эта инструкция будет идентична инструкции `insecure_withdraw`, за исключением того, что мы будем использовать ограничение `has_one` в структуре проверки учетной записи

 (`SecureWithdraw`), чтобы проверить, что учетная запись авторитета, переданная в инструкцию, соответствует учетной записи авторитета в учетной записи Vault. Таким образом, только правильная учетная запись авторитета может вытягивать токены из хранилища.

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

# 4. Тестирование инструкции secure_withdraw
Теперь давайте протестируем инструкцию secure_withdraw с двумя тестами: одним, который использует walletFake в качестве авторитета, и вторым, который использует wallet в качестве авторитета. Мы ожидаем, что первый вызов вернет ошибку, а второй успешно завершится.

```javascript
describe("account-data-matching", () => {
  // ... (неизмененный)

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

Запустите `anchor test`, чтобы убедиться, что транзакция с использованием неправильной учетной записи авторитета теперь вернет ошибку Anchor, тогда как транзакция с использованием правильных учетных записей успешно завершится.

```
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS invoke [1]',
'Program log: Instruction: SecureWithdraw',
'Program log: AnchorError caused by account: vault. Error Code: ConstraintHasOne. Error Number: 2001. Error Message: A has one constraint was violated.',
'Program log: Left:',
'Program log: DfLZV18rD7wCQwjYvhTFwuvLh49WSbXFeJFPQb5czifH',
'Program log: Right:',
'Program log: 5ovvmG5ntwUC7uhNWfirjBHbZD96fwuXDMGXiyMwPg87',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS consumed 10401 of 200000 compute units',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS failed: custom program error: 0x7d1'
```
Обратите внимание, что Anchor указывает в логах учетную запись, вызывающую ошибку (AnchorError caused by account: vault).

✔ Secure withdraw, expect error (77ms)
✔ Secure withdraw (10073ms)
И вот, вы закрыли уязвимость безопасности. Тема большинства подобных потенциальных эксплойтов заключается в их простоте. Однако, по мере роста вашей программы в объеме и сложности, становится все более легким упустить возможные эксплойты. Очень полезно привыкнуть писать тесты, которые отправляют инструкции, которые не должны работать. Чем больше, тем лучше. Таким образом, вы обнаруживаете проблемы до их развертывания.

Если вы хотите посмотреть на финальный код решения, вы можете найти его на ветке solution в репозитории.

## Задача
Как и в других уроках этого модуля, ваш шанс практиковаться в избегании этого эксплойта безопасности заключается в аудите собственных или чужих программ.

Уделите некоторое время для обзора хотя бы одной программы и убедитесь, что применены правильные проверки данных, чтобы избежать эксплойтов безопасности.

Помните, если вы находите ошибку или эксплойт в чужой программе, предупредите их! Если вы находите его в своей программе, обязательно исправьте его незамедлительно.
