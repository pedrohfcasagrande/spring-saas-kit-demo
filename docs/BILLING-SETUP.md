# Billing setup — ligar os planos ao Stripe e Mercado Pago

Este kit traz um catálogo de planos no banco (`plans` / `plan_prices`), mas os
IDs do provider (`provider_price_id`) vêm como **placeholders** no seed:

- `price_REPLACE_ME` para linhas Stripe
- `preapproval_plan_REPLACE_ME` para linhas Mercado Pago

Em `prod` ou `sandbox`, o `PricingService` loga um WARN no arranque listando
todos os planos pagos ainda nessa condição. Enquanto não forem substituídos:

- `change-plan` falha com `IllegalStateException` para o plano em causa (precisa
  do id real)
- `checkout` continua a funcionar porque é feito com `price_data` inline, mas é
  recomendável migrar para price IDs para alinhar com o Customer Portal e com
  o Billing dashboard do provider

## 1. Stripe — criar os Prices

Para cada plano pago (`COMP_*`, `MEMBER_PRO/BUSINESS`, `CLIENT_PLUS`) e cada
moeda cadastrada (`USD` e `EUR` no seed):

1. No [Stripe Dashboard](https://dashboard.stripe.com/products), cria um
   Product (ex. `{{APP_NAME}} - Starter`).
2. Em cada Product, adiciona um **Price** recorrente mensal com o valor em
   cents da linha correspondente de `plan_prices` (`amount_cents`), na moeda
   certa, e intervalo `month`.
3. Copia o Price ID (`price_abc123...`) e atualiza a linha no banco:

   ```sql
   UPDATE plan_prices
      SET provider_price_id = 'price_abc123...'
    WHERE plan_code = 'COMP_STARTER'
      AND currency  = 'USD'
      AND provider  = 'STRIPE';
   ```

4. Reinicia o backend para o WARN desaparecer.

### Via CLI

Alternativamente com `stripe-cli` autenticado:

```bash
stripe products create --name "{{APP_NAME}} - Starter"
# retorna prod_XXX
stripe prices create \
    --product prod_XXX \
    --unit-amount 9900 \
    --currency usd \
    --recurring[interval]=month
# retorna price_YYY → atualiza plan_prices
```

## 2. Mercado Pago — criar os preapproval_plans

Para cada plano pago em BRL:

1. Cria um Preapproval Plan via API:

   ```bash
   curl -X POST https://api.mercadopago.com/preapproval_plan \
     -H "Authorization: Bearer $MP_ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "reason": "{{APP_NAME}} - Starter",
       "auto_recurring": {
         "frequency": 1,
         "frequency_type": "months",
         "transaction_amount": 499.00,
         "currency_id": "BRL"
       },
       "back_url": "https://app.example.com/billing?checkout=success"
     }'
   ```

2. Copia o `id` da resposta (`2c938084...`) e atualiza:

   ```sql
   UPDATE plan_prices
      SET provider_price_id = '2c938084...'
    WHERE plan_code = 'COMP_STARTER'
      AND currency  = 'BRL'
      AND provider  = 'MERCADO_PAGO';
   ```

## 3. Verificar

```sql
SELECT plan_code, currency, provider, provider_price_id
  FROM plan_prices
 WHERE amount_cents > 0
   AND (provider_price_id IS NULL
        OR provider_price_id LIKE 'price_REPLACE_ME%'
        OR provider_price_id LIKE 'preapproval_plan_REPLACE_ME%');
```

Lista vazia = tudo configurado. Reinicia o backend; o WARN deixa de aparecer.

## 4. Customer Portal (Stripe)

O endpoint `POST /api/checkout/portal` abre o
[Customer Portal](https://docs.stripe.com/customer-management) do Stripe para o
`providerCustomerId` persistido na subscription atual do payer. Para funcionar:

- O payer precisa ter pelo menos um checkout Stripe concluído (o webhook grava
  `provider_customer_id` em `subscriptions`).
- O Customer Portal precisa estar ativado em
  https://dashboard.stripe.com/settings/billing/portal (features: cancelar,
  trocar plano, atualizar método de pagamento).

Mercado Pago não tem equivalente — para cancelar/alterar plano o usuário usa os
endpoints `/api/checkout/cancel`, `/reactivate`, `/change-plan`.
