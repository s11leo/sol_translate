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
