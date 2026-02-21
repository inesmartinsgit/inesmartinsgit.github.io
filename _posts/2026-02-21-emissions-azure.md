---
title: "Why Your Azure Emissions Dashboard Isn’t Showing Data (and How to Fix It)"
---

*Disclaimer: Everything you’ll find here reflects my personal views and is not affiliated with Microsoft.*

<br>


As sustainability reporting becomes part of executive dashboards, organizations need reliable visibility into the carbon impact of their cloud workloads in Microsoft Azure. <br>

The Power BI **Emissions Impact Dashboard for Azure** is often the first tool considered. While technically straightforward to deploy, its billing and permission requirements are frequently misunderstood. <br>

Verifying permissions may seem trivial, yet many teams spend hours troubleshooting the dashboard, only to discover that the root cause lies in a misunderstanding of Azure’s billing hierarchy and permissions.

This post shares practical lessons and a proven workaround drawn from real implementations, helping you avoid common pitfalls and accelerate your carbon reporting setup.

# Understanding the requirements

The prerequisites are clearly defined in the [Emissions Impact Dashboard for Azure](https://learn.microsoft.com/en-us/power-bi/connect-data/service-connect-to-emissions-impact-dashboard) documentation and appear straightforward: 
- A **Power BI Pro license**.
- Appropriate **permissions at the billing account level**.

In practice, the main challenge usually lies in the required billing account permissions. <br>

This is particularly true when the person configuring the dashboard already has some level of access but is not fully aware of its scope. <br>
In such cases, attempting to configure and load the dashboard may result in errors. <br>
In more confusing scenarios, the dashboard loads successfully but displays only demo data instead of actual emissions data.

To understand why this happens, it is important to briefly review how Azure billing accounts and permissions are structured. <br>

# Diving into the billing world

An important concept to understand is the **relationship between billing accounts and subscriptions**:
- A single billing account can be associated with multiple subscriptions.
- A subscription can belong to only one billing account. <br>

Billing accounts sit above subscriptions in the hierarchy of Azure.

If you are setting up the Power BI Emissions Impact Dashboard for Azure, you need to be a **billing account admin** for one of the following agreement types:
- **EA (Enterprise Agreement)**: this means that your organization signed an Enterprise Agreement to use Azure.
- **MCA (Microsoft Customer Agreement)**: this means that your organization signed a Microsoft Customer Agreement through a Microsoft representative.
- **MPA (Microsoft Partner Agreement)**: this means that a Cloud Solution Provider (CSP) partner manages the account on your behalf.

Regardless of the agreement type, the billing account must have a **direct billing relationship** with Microsoft.

## Common Pitfall #1: The Wrong Billing Account  

A frequent issue occurs when the person configuring the dashboard is a billing account admin but not for the billing account that contains the subscriptions they want to analyze.

In this scenario:
- The dashboard loads successfully.
- Data is displayed but only for the selected billing account (not expected).
- If that billing account has no emissions data, the dashboard continues to show demo data.

This can be particularly frustrating because no explicit error is shown. The user believes they have sufficient permissions, yet the expected data does not appear. <br>
Azure billing permissions do not always produce explicit authorization errors. Instead, certain menus and options are simply hidden.

## Common Pitfall #2: The Global Admin Misconception

Another common misunderstanding involves the Global Administrator role in Microsoft Entra ID (formerly Azure AD). <br>
Users frequently assume that a Global Admin automatically has access to all billing accounts. <br>
However, billing permissions are separate and the Global Admin sometimes cannot help further.


## Recommendation

The most effective approach is to contact the individuals responsible for managing billing in your organization. In some cases, this may be a CSP partner. <br>
In large organizations, identifying the correct billing admin can be challenging, especially when multiple billing accounts are managed by different teams.

In parallel, I recommend the following workaround.


# A workaround to unblock you

The Power BI Emissions Impact Dashboard for Azure requires permissions at the billing account level because it is designed to provide organization-wide emissions insights, typically for sustainability or central governance teams. <br>
However, there is an alternative that requires only **subscription-level access** which is often sufficient for engineering teams.

This alternative is [Carbon Optimization in Azure](https://learn.microsoft.com/en-us/azure/carbon-optimization/overview), available directly within the **Microsoft Azure portal**. <br>
It allows you to review carbon emissions data at the subscription level without requiring billing account permissions.

The main limitation is data retention because Carbon optimization provides up to **12 months of historical data**. <br>
If you require emissions data beyond that period, the Power BI dashboard remains necessary.

Additionally, Carbon optimization can help you identify which subscriptions generate emissions. <br>
You can then map those subscriptions to their respective billing accounts and engage the appropriate billing admins to obtain the required permissions for the full dashboard setup.

# Conclusion

Monitoring carbon emissions in Microsoft Azure is becoming a core part of governance and sustainability reporting. <br>

While the Power BI Emissions Impact Dashboard for Azure provides comprehensive organization-wide insights, the main challenge often lies not in the technical setup, but in understanding billing structures and permissions. <br>

By clearly identifying the correct billing account, the billing agreement type (EA, MCA, or MPA) and the appropriate billing-level role permissions, you can avoid the most common pitfalls and save significant troubleshooting time. <br>

If billing access is not immediately available, Carbon optimization in Azure offers a practical way to begin analyzing emissions at the subscription level and can help you identify the right stakeholders to engage.

---

References:

- [Emissions Impact Dashboard for Azure](https://learn.microsoft.com/en-us/power-bi/connect-data/service-connect-to-emissions-impact-dashboard)
- [Billing accounts and scopes in the Azure portal](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/view-all-accounts)
- [Carbon optimization in Azure](https://learn.microsoft.com/en-us/azure/carbon-optimization/overview)

