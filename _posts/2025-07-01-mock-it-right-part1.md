---
layout: post
title: Mock It Right!
subtitle: An Introduction to Testing with mimic++ (Part 1)
gh-repo: DNKpp/mimicpp
gh-badge: [star, fork, follow]
tags: [c++,mocking,mimic++]
comments: true
readtime: true
mathjax: false
---

* Table of Contents
{:toc}

# Introduction

Writing unit tests for software projects can be challenging — especially when those projects depend on many external components.  
A key strategy for building maintainable systems is to minimize coupling to such dependencies.  
This often involves the use of *interfaces* and *abstractions*, which allow your code to remain flexible and testable.

{: .box-note}
**NOTE** In this guide, the term *interface* refers to a general abstraction and does **not** necessarily imply a polymorphic class.

We’ll explore the fundamentals of *mimic++* using a simple quiz-based toy application: **Mock It Right!**.  
This app provides a hands-on way to learn how to create mocks, define expectations, and test behavior effectively with the framework.

## Mock It Right!

Let’s begin by examining the design of our Quiz app, which is fully contained within the namespace `mock_it_right`.  
A quiz consists of multiple items — each represented by a `QuizItem`.

Each `QuizItem` holds:
- a `question` string containing the quiz question,
- and a fixed-size array `choices` with four possible answer options.

Importantly, the **correct answer is always stored at index 0** of the `choices` array.

```cpp
struct QuizItem
{
    std::string question;
    // First index is always the correct answer.
    std::array<std::string, 4u> choices;

    friend bool operator==(QuizItem const&, QuizItem const&) = default;
};
```

The main logic lives in the `Quiz` class, which is initialized with a non-empty collection of quiz items:
```cpp
class Quiz
{
public:
    explicit Quiz(std::vector<QuizItem> items) noexcept
        : m_Items{std::move(items)}
    {
        assert(!m_Items.empty() && "No items given.");
    }
    
    // ...

private:
    std::vector<QuizItem> m_Items;
};
```

`Quiz` provides a `run` method that starts a quiz round.
This method is responsible for:
* displaying each question and its choices to the user,
* gathering user input,
* validating each answer, and
* returning a result summary that reflects the user’s performance.

Because these responsibilities are diverse, we apply the *separation of concerns principle* by introducing a dedicated I/O context interface.
This abstraction handles input/output operations, enabling `Quiz` to remain decoupled from specific I/O mechanisms like `std::iostream`.

To express this I/O contract cleanly, we define a C++20 concept `io_context`.
It ensures that any compatible type:
* can read a single character of input,
* can display a question string,
* and can print a list of answer choices.

```cpp
template <typename T>
concept io_context = requires(T& ctx, std::string_view question, std::span<std::string const, 4u> choices){
    { ctx.read() } -> std::convertible_to<char>;
    ctx.write(question);
    ctx.write(choices);
};
```

{: .box-note}
**NOTE** Not familiar with C++20 concepts yet? No worries!
This just means that any implementation must (at minimum) provide these three functions.
I’m only using it to show that *mimic++* works even when there’s no inheritance or virtual functions involved
— something many other mocking frameworks don’t directly support.
No need to fully understand the details — and I promise this is the last time we’ll dip this deep into language features.

The `Quiz::run` method is then templated over this concept:
```cpp
template <io_context IOContext>
std::vector<bool> Quiz::run(IOContext ctx) const
{
    std::vector<bool> results{};
    // (item-processing)
    return results;
}
```

It loops through each quiz item:
```cpp
template <io_context IOContext>
std::vector<bool> Quiz::run(IOContext ctx) const
{
    // ...
    for (auto const& item : m_Items)
    {
        ctx.write(item.question);
        ctx.write(item.choices);
    
        int answer = request_answer(ctx);  
        results.emplace_back(0 == answer);
    }
    // ...
}
```

We expect the user to answer with one of the characters `a`, `b`, `c`, or `d` (case-insensitive).
All other input should be ignored and retried.
This logic is handled by the private helper method `read_answer`:
```cpp
template <io_context IOContext>
int Quiz::read_answer(IOContext& ctx) const
{
    int answer{-1};
    do
    {
        switch (ctx.read())
        {
        case 'a': [[fallthrough]];
        case 'A':
            answer = 0;
            break;
        case 'b': [[fallthrough]];
        case 'B':
            answer = 1;
            break;
        case 'c': [[fallthrough]];
        case 'C':
            answer = 2;
            break;
        case 'd': [[fallthrough]];
        case 'D':
            answer = 3;
            break;
        }
    }
    while (answer < 0);

    return answer;
}
```

Implementing a main function that connects all these parts is beyond the scope of this section.
If you're curious, you can see an example [here](https://godbolt.org/#z:OYLghAFBqd5QCxAFwE4FN0BoCWIIDGAtgIYDW6AgqsAM4gDkAtACIDCAspQNICiA%2BgCEAqgEkAMi34AVAJoAFXgFIAzCxboARgFdgDLAQD2RAA44ANulTiSAO2DaSwdKIAmIAkoBMg74IPatMjGAPLayCbhAGIW6LYkROggAI7aOABe/MjoQbRYAGax9LDorjjBqG4gAGwFsVUAjAAsdZbxiSCkOLYAdAQmJligDPo4tKK2BObarqWMDaO0HCTdAMqG2qgESQwLuLQhJnGMAAz65nbAVSMAlFgQpeWGle4NrS6vAKzv7Umc5OhxGNkLQesgAB7IIaMEyoQwAK3QBGQEFoCBIR1oNyUJ0oOMoxAB/CI3RwRG0RH4GFSOAwrggADVeAAlVaiEIAOQA1Coep9sbj8SRXK5%2BOhwUjwiRNJZUejMakSBh8VzVVy0RicgB6Ai0qboPoDFVqjWYrUYZGXSyGkzG1VdXr9EwCvG45BK5zIfjmbpkb04TSoJU4HJyzW0RXK3Fqrn5IjIVwuxYTKYzOYMM77ZZrDZbHaZsaHY4Z86XKqEih3B5lCpVJhvQqWRoATh%2BCSSAEU0ukbdCGN4VN1U7Muao2PkghgEqplIKvIPJtMR2PJ91gDP8figu4QKv7FzPfxXCR3RAbqOAOy%2BaMmhMgEA4CdodAJLmP2gQbxeDC0XPbWhagg6DmOYhhgpCX7Yio16UDGgRrlyojIKOajqneD6Ti%2BRA6Pk/DlFYJ7PGOBDoqgG7QXaXIYMgmy2GhO57sAEBIRA753IhKI3FBMFKBeLD4smi5pu4JZZistjrJs2zzIsRa2DJWAXPY5Z%2BH4VaPLW7gtI2HwgF4Jxth0XYZD0CADH2gnDumBZLOJkl5gphZHPJolKVcIkMOpNbPFUXjvL5fk6b8KA5CCvbDJZS7WYs2YSX%2BOx7E5xaZm5Kk%2BGpdxjP8FDyHCiLIqc%2BjwiAby2OgADuMRNu4853OYjCfPoLmZoYjDSBg6CjvOezIIVXFYGQeknHyTQXs2TSfPpnyjeN02KYwTRNYVWCtSMWD0AZvUllWMBQLtEBIEYsy8N5qC4PgQXtiFuS9vkhi2MgqwECQlgldp93IMItDoPI4LLcVfluR4qk%2BOtwFIsgOD3fgcSuGwhjTEQ8nVLUsNAmVHIUpoVhvbUJiGLQ5RQ7Y8OI8jeME0T93o%2BgmPYTjzS1N9ljIsTqzuqgyCkxS5NgyzkP3ezSrIDTdPY6guPrRzXMIzzNRM9LotYwzTTVHcv5STsA5DlFKFsM9yAkV4OoniRWShcSJAEHCoJmbaKizni8462mesG0bWrOGVQYVAB7sIPwXsEb7plGg7m7O0Jy4qPrpsIMbpCG0BqB%2B3HltJ1YtBUpc6Bioq5i22Hjva1HnVjiSJKeOlPhahXOBV74Pih/bxeR1Zet1w3fhipCWrChi2Qp1qbBx14zcboKc4Lu3X7GT2dtfpuuK/LQJhW%2Bgm5XpR8H7kQhgEH65RUjgwAIL1IBz0h6BEKoMExiva/bFyickTkKEsM/ZL1wM94v0BtC30og/deXJxRHGRO/T%2Blcf4gDARDQBN5VTAKfoUeIIEACekDO4wNQS9cw6CEGulgreVA2gIHsnhg9cUyAOD7zIJRXid8YxqmwSYe8tCD7EVIlyM8M4qIvlcIwxhPRfiQK/BQj61COGDRAFOQRXgvC8X4nxQhzCWFfydOwuhY5KJqNVAAN0MDgVwPDtz3kYvwfRIZyp3F0XowxxjTHoVXnYFczi0BrksdYrkRhbBBCwFyJo2gZxQV4FycqqB8JCMQXotRwjRGqA/uIkIlDsiQmkfeCJUSFFKNyRRGJeSeIxMnGQ5CzI7CuGMBkKwUQAR2NVCALkkQZT12fnQvCXpImn3PuU2wlSSTpCsAwreMSYxmJAPoiGREY6X2yEQHx90ggACo%2BFom0PkfIspxkuNsGOWZ18Fl%2BOQHw/CRAsRckMJM1AkTZj1MvEw2JqotRai5AAKUCMhaitEuTICAvM7oPygLqnbBc1AsxUA9DuTGL5qA6KnIAaMx5qpeJsHoveKxFV6AoCDH4u6qAiCfk%2BL4T4LAIAkHCIYQ5QRvDVDfOeRh/CaKwq6tUHAt9LwsBdEiuJF5UXjJxc4LFwQ3E7kmciaZvBeH5OITy/ik8ZXMJIkY7Y/A1kbMsFSHI2hzDITVZs9AqJnFr12THbZHj9y%2BP8YE4JDsfEIGVTkc8lyrA3I3oi5FIyFWxKVfXXOeqNU/m1Z8rVOrhFGGuRDCYsxwSQJOMElR0ruVotkTnLFRgTDoMIPa31eR%2BG0CDX0bN/4ejY2AN0M83EoVqhhXRQNobE16NyQUhNPEVHyukLwVY0h%2BBsEoKsSVs9uz8KMMAWw1TaALIjciMqtAJ2GHyAeHAky6J2FoOVKwORIUKJdNEhVe8D4dOPt0%2B8c8qXIRpOkXdsTGH7JvleHoqRQrEzEQojsDQvwBMLQ6idiT7ldS8JQd9CiAlfkoIo4D/7KAqA/ZBpoi8VEIZYFgO54zSAUH4NoMdj6xy9P6dU1AtSKAzl4S2peCryXBDPaOIlcDkSlHhlO5AM7c2rvXWRElkCADivAOQskoB2z87qfnSksGOTQhgEYBJIkqYjV7uWMPyC9b6ASwAMEEKppRyGhMxgU0p7AXJVMEA03xLTXrG1XkUwXfTqnXDGaQ1Wj1gg0DaGswwSgdnTNJsYc51zJA7N3KUVxVRapKKsPvE9QwRxXCrHQI%2ByYnUoYxbi9sIRpGYkSKoekuhb5DBsAhKl/iDbVSJdiy5%2BLXVfCoShnl8EPQsnZB6LRr0z0QKflfUBrwlaYkleS2XdKVXcsQjq5EhrTX%2BAtfMBAP%2BOR7wCqSLA5IhqdxKiDAQreCjAMwdA%2BBvykHoMQdA3BnJfEgvSsos8rkAB1ICdF11NLhFYkcdg3y2H0S9RxrGrABLu%2BVCw5gDzoGQnYQwvyrB2rsNsSFMSLviEB6pideKn6/LGICjqPyyRvwBSQKihhypgDAJRHrZWn6qQGzVnocjGvgnAc1vBvChMVNAdTiG95IaJA/CoLlejGe4IwbNwHtEPyqYABqqa60QmMMO4cMAnbYXH93LnGM6rmLkVsaIvVV34tjUOFVE7iCT/rH9qtDcp2Nib9OzOqh590PB6D%2BdMr8WSrXVhxeUXGWKioVG63IV/ReinmHWKDfBK7mJzJeAdjEGHiADRtDv1/d7nohNBkVuC6qNgAAJXgbBuAPGZ3RuGzwLRMZyD%2BtQ8eQ3IB6PkOED0U9nbbUQ5Bbq8SerGc5iBuGqmDNQNIu5jTmk%2BgIG0g9R8uln3vJ3gZQzm0POYRwFJ3B%2BAhCZMycQIRKAsF4FIDgvBpDp5CKShz6oEDrP1Z57lG%2BpDL5ZGvjfS30VTLIjM7sV95mWuQCsh2AT7%2B7mNXsl/cyZ6ISti2maol%2BS%2BK%2Bt%2BpKPqKq/quc3u3%2BP%2BOyIq5i5qwAZ6ASQSwBp2s%2BhSZGMYcyJgFw2Qes2y1SWQb4E8lu/CNIP4PCOAes1qXO9o7So%2BJ84%2BIAk%2B%2BG94MBfqJ%2B6q8BFeOO7o2Q42RaoY2yf%2Bpq7ikSFqiyUI1qfCvBWIwys%2BMY%2B6h8nS7BPSFSXeOMHg4hqq/B%2Bqmq%2BaOqeaQaYahekafS4okCEAQSTAOA9KXgnwihaW1B/Kqas2IOJ4uc6amayhUm4hoIpa5aLhggb4ASCeyhJa6AZatgdeah1aAuzK3uqeHKm8cqRCHaXaPafaA6r6Q6cBE6eCb4cypRfSgK6AtImua6mcL2ZQj2jgGCsYzwoCVsCAVEmGW6nWm8s%2BO86B02WK/AhCbu6EHu7R7%2B5R18MWxyreeit6wij6QQz6v6s8HWn6yhYi62XgbAmx/6l2DQ9YW2CiRxTAO2IGZxxx%2B2iiiG5%2BaiSx96KxAsdE6xr6lxXIX6OaOxjcexnxX4l2XgJxB2ZxwJAJYJTAtxuSJmdyTxggD6Lmqx90L6XgHYtxWxIRvxX4bAGJhxKgIJu2gJBJEJXgl2BJ0JiGwyhWRS5GFKFy5UZUrgXB3ekCqGRImGOA2GMcLJVgveDsUqs%2BFGhgNKVEuhU%2BqAkCSyuOTJvJT%2Bs%2BGhh6Y%2B58p60xF6jCpycxASqGzqEAMppQcp2IHhoWGiMCEWUWSWxOnUOKeG3eSWBWGRF2vAeeyEoOL2kQrplKbphQKcyEiexhlgPiZRlG2MoC%2BcPylKKupIkMGumpgO52LyjO1sL4JB2OX4cB8iXgFy4QHpXImgmChMpg%2BCwhJ4CE90nUlGbplg%2BQvUCZ1GggOABkOAbwOAiiHGiSfCSgRKzZuAfkjZXZ2R0K4p%2BGSWFW8ew53e/pp%2BGqVONO42dOwxs2Oc94sWrEcycxQWQmF2NaWcGIRB6CphQa/A86Y4cafC8IHyw6NRkyE6bpcaqupRNAo4DOVRvO%2BC9ugu/Ae5%2BCh5Oqx5%2BQp5NqA6oByKRKA5ZK9J7%2Bop8KRpyRSKwpTSaApeRuFRR%2B16vKyaGK5UQqOKtAeKBKXZxKpKCFUFXgtKzhf6NaLKbK0EHKzBSazCKKyac2QqIp0hoqj%2BxG4uDFt4O4LFPhIhBqJgSFASwlKccRCRZ4FWXIDQ3FPFjK3yYlACRWjygWjpLylAE6d2cu5UXIZAOl1RoK4KZZC6UZFRbRkp%2BaGADxXIF2d2aIGw5gJiSmlK1IaQGAhlU5AhQZIECO7RZRpyTA2xAKvyJ4RlVgOuQ5fSehqAo5pOH8NpMVXlJhs5EM85rWi5Ka9g82q5cZxyRKMeA53x/4m51BF2UQ7RmGSomC%2BQmGrMiy0RqRfiX5AwP5eCv5Xo86b4E6pWS6L0cQnpClsKu5rVB53u/5gFM4kV3Or5NufOsiTVI1%2B5X5IEHV/5bW1QcIglYh36Y4hVDsclqoiVEpcVhuYp0VEpyVM5ZuC5ccM2WVzgK5i2eVhFXgwSJKxVjq9FMY1uaC75C1DuS1P541J5McZ5ApNKW1fhO1OaY4b1ISGRx1I5sWY5qESNk5cB/AqVyI6Vk2mVc2T1a5sx8ZRKZ5H1yhpVsSv1tuH5w1LVy1INAFYNQFG1UNohyhk1B1YxMSYWIA5ppQlp%2BuCWhggt8WDpKlGWaSNC2Wxu4I4ttJMYeu5W8VOW5O9WBoN1rWs8O2h1tlLysOyA8OFl5IxByumwTONOpQ7p4QeZmC6IVRycBohOItpWQtqNRuQew2%2BE2NtOWtQm%2BNy5C2P%2BwQX51yJABC7FqBshwAXiFUsmuxQJhJVxZJ4JpxKdUJadbAO2gWOBdZl21pTVmuL24aRedRbG%2BOztotBulWHt5OpuLpuNFuVNs1f1duANguEAqm6mnk3NuuLtvW7tqtQ26tPtjds8txutSt1dv6stXto2Dd5udyAd2VhN4yIdK24dKBu4aBsd5U8dfx5JFxadh9eJOJp9ZxBJHWOdutUuhtMuXIBlVFbpJdEMZdEVldrtytZ1s99dc55u31ao1N81O5ndDAbAYuvdit/dVpg9s9I9mtk2GxkE6lV2nU35IYzlJZJBXVca32CA9cXRlgJAN5oCVy6CyO%2B4JAE65Q3V01xW0DbtKtcDI2GtC9t1GcWKBNQdL1pNRVFNN9Ly%2BdQ1K6cKkw1hECn2qAFd3WDDX9NdQ9tWv9aV/9dyQD/1IDqm4DPdKl4xHF4qkp0x3ukCfupCiRstutlEYeEeogYeMg6e/Ggm1BCBdyI8Sc7Cd1KcE%2BOcvA%2BcH47unFFmemASPmASlm30OdYxDenk80DAjUWAzU%2BgrU2QN0ToWAW0ZwVYh0hgx0p050EAl0HQDot0H0T0/Vb0BQH0X0P0f0okxUtQQMXcoMzMEMxMMMfS3MSMIAk0WAaM3QtMysEs1QBk%2BMhMrxHT8k3TIzVMtgSs9MgzBkzT9VEk0s4zXTfkizrxQsnMsz4sNQCzKzssnT3TQQwsOzOMQz6s8UjAJcM8RRJkC8OS8qNzusY4L0wAzw5QCAN84cU8Ls0c44dVrxL0VBzzrsY4UMmE04PzTs08LzMcx1ILbccLbALFILy87YLiKWgoCxqo/j%2Bj5crBWhx6F8AB18fCywFAr%2BcxTdIWOLzCgxw%2BmhR6HBSxKlbeejFQ/%2BGQr%2BJy5l6xdLsS8JaFaoiJT6KJ7xZJ6IyEKObpswimQaque8mGChz2TWpZKJ5IQQz8ccqgUQH6wrqon1yFf6X4IQYtoJXgHI6AVyad0gv2KWFrTQWZbOOQ8Gg58msJIFXIQrXrMYoryJbxqE4it2%2BDJE4VkpmryEarrxE6oZ02rgvRNlSaRr2JvrjaCiHIkZYKVgCEbltIpQibBrjFCiEwXIswUwSo6rdEzw4KhbabcSJbta1rmcnU5bFwPsz6Nb79wGRbOmCiIQ2b0duri89bmRXrTa1BeB8qsSVF8KGRE7C7uI3QyEDoFucmqoFbs6XIcpfeTS2gLSQ%2BipbBxLO7M%2BlEA%2B9cIAdyDLt6WChLzLKppL3zCteieLnLz%2B3LgB7%2Bn%2BYScBSBUhbAt7UFtqMFFyVyrqAWAr3O9JSlkCc7o7TF7u1iOFq6%2BFhKRFEFlGpF5F9KV4wj1F7KalCHGFXh2VrFW9kxT%2BkqutsSpHgq5iAZQlIlz8/AzI9gNHeiVFSlGROmDedyvBRh05ghZhuqjH/7rikd290dGBihtqFNYHLqSukHcFiqhhmNRj6RW8HhSadH91ARWa36jVInRrEl4RIe1BkugjnUcBquIEdqhnTO2wwlhlPpWrpydDjyF2IQoOqAv2304SnU5UjlJioE/naAVsZAFyC6z94j0rcyThthtWR%2BunWKf7MRWJiHExyH94rgcIJg0e7ElIbHwA5nSat9RtBle8kyhlL9ECvBEZqu51/SjR4oHnM7FefQsXUadhv6RX9gUl3gbhsRSeBqHHaiKX94%2BEqAqq5UGIEA6X36pniR7EC3OaS3A3PgFhOqnXjG3XwekD3KVF6RbLtL2R57kSb22QV7QmqGyADQzYzYKgF4LHxXjC4y3QhiFAP%2Bx1R41rvqBWRpNJ1JB3LBI%2BRLLLQ6ap3YjCFLLg65gOZ42p6EaGucnJ3JbAO7ApgPIPaEpC5CKSkiEEZ7Qm0mkpciNLPKKn7LPBGwPuMcY4/6vAVCkp6CKukjjSi8J3aipPPiPHr76EuogbygcnfP43AvtPesDPi8nwbAuyOSXPzCVFngCvi71BDiJi6tSBO9WFXILxxMADjmyX4vNtOiMcuvSJrxkvZv0vsvnPKnqvsS6v4SLD4nJqbAZq0du9MnWBcnIRBv9yRvO4RgJv9P1vG2HPCiVvqKHNvD7ZofqKNvyEggEfWZpv0fIRhF76cfbAUvOSMvcviio7Om8f/6bAKfUf9nsNRKbZSSJfifXILA5faflf/4hF0G2fufbZtv8v9vDevHQP07aokt1CCj8tujD%2B%2BjXuFexrJjAe5jOPJF8hEZ7o5gr%2Bxr3uoII3gplEi/Ryk6RelAzuKcbJ6E/FHgGwteG/wTpC6Aut4ywfdPOfZvtXyAh/9Rx/zfX42ZyEXVX/zfwQK/NfhXy/6SMtKG6ffhDF6KRNsi0TeqLEyWiiRWoRTVJuk2iYHRz%2BOTJ4GdDwD5NYgwUbKICGBCggIQUIO6A9FKavRmgFTB6FU1%2Bj/QSoikS4B4BR58wWm0MB4O00ObyQ/IvTDGAMxKhvApmYzLgXpCwBCDiYZzCWA0DeAbM2YBzMmKINkGCxFYfTMWAzBkHyC5Y6zFQXwLmYCDLmmsRgBWFzgkgx05ISkHmzpCMgWQbITkDyB6D3cd0uIWEAiAhhtYvA0iDiNu20IV1t0ZGAcMKFFDihJQ7oGUAagvT00I4KgFhOJBtBRCnBlAQIT3BCEiZwh3Yc2LkEojJMwoTofEAkIHBcgvoLRfBAEnLLhJguXIQIJ1FQSihH4ZAJwAalYTbtw8keLfC6CSH5pNAZQIvM8EzRfgtQkWZADqGMBmBLAqAJgGAlAgYBUAWoH0JoAAisItQzmWwPQkj79C9yWoHQBYFcC1xTStoPwW6A9CA5/QKw/0IGGDChgIh2QidPIGZCiAGQ/GR2DGFcZGxYogiGJC8PjjvC90ew3%2BHsLyECQ6oDUBAS1BAAECgQN0EgWkz6j3AsmmAioHkwKadhuwzcagY9BaxJAqBZAz6N9DoG1MQALQBpiDH8BKD5IHAgvAoK8AXgemfSSQSAAAAcggymMIMpHUjxB1MVQfwMZGsClmWzGWKyJ5GbMdB/TPQdyJOacxVmVIqWKc05GijZK60K5v2Gdj5BbCC6axgAC1%2BA6eKIXKz6besxAmo7UU8yRZgsY4G9COq3FhamjY4s6KwMcmhagt/miQPeKgAtFRC/mZcaQkOwdEmj/mFiLCoiytH/NKOgYj0aQSkLFx0W7OR%2BJ1CPbg9jk2LWfCUggRLFienhGQghD173RReetLkDEF9ItcY0KOF6LN3QS3kgUL/N%2BhChQzoRzRW9RiJgSAot9XWZ2ITNXhDBVFxMCMC5EcB9jTJCspKIDvIRpQBIhxRyGlPSlQhytyU9aXAsaRiSEFTaesZAOgmcggppAVBGML4ic7StDA42QnnTwSq9UfwEADcWRR8QQgkeDEbXt4izG2Arx5iADh708Q6938jYpQn73XaMU8OyIRRgIgG58QuQTAVZOL1ex2iAwGqYVDHFJ7kQVOv4uemkLFa2AxuF42rJrwprzt5xCqTdhOlPYKovxF7AgNdzV60h1c/2AAH5ykBuU49APK1nH4F7EpElophU4oft0gr%2BM9D%2B2PxCdXeXLdiV%2B2HEgcKik4j%2BCcBx5t48eyEATupwrzKdA%2B94OsZJ39HWJ3xvvXagr2p67hyCUkrrolxzFTsSJnMZidJMY4dVuJAhXiUpLQLe8mx8nX9GJJ0ZYStxFwLdnPFULnt92g%2BYibEkmH1waGc8H/CGLYk8shJ18XNOMjR4uZ%2BAYlHDBOSGS2p0aLuB%2BoYHFA7ij8jSSkGvze7I9dSMFTTEfiK5xSyIV4HUpMnm5FTseQmL8VTVtGcxO6YATKRUUaymAVxG3aoKKS/CZsZiZyRdMul6IoSfqtUlEIVIur4YWUHUjNtqn%2ByJTJSZaPqcgxV58chMi4vwnrChj7jMsyEYfhBGha0cJirEtgF2PMB8JTGzEAnptNFK/jzwUFKqVBzF4cspiS/eFFglY5FTgJDsP9o1LCkDTeKE/d9odIkzHSEp0/MfoPyRR4oeEu/ZZN1K5CNI8p47O6Y8ihk%2B4iUQRCAciD24DkXpcpd6bwD/aBUZwGEjSXongma8CZDsO8T9O57D0XeRMl9jxWXZVjjGLmV0V%2BSP6EAIQVMxXtP2alEF14/ATQBF0IC6To0ceVCJIy5kO9DuhdDfvpKWkKpYQfVK7jWIelP5AOT7PhF9LOQ5iIpWGKKTFJ5JFTNZr00ad3nElqhGZblC2JIzOmpJqEl0zmUAVulU8LZD0KsYwhOLaduUlSI/NVO5RrpygYbDmX%2BOFAp5iOLs7nlQ06iqY/MDADnmBSJSWYdUCAOELoAQADkM5xMyOf51UzuY45vbQBkf1jQ5ik0gYF8PQizmKoo5BmBgJoFUzxziUggJOb8lTmnwM5JKEuSTOrld165Bcq3EXN/TvpK5sSMueQE7nZzo5DAIzHHPrKEVm5KcjYG3I7kdzh5aoZ6DnLAa9yi%2Bhc9/pAkUSry1Eo8iuRHKrkbzbMM8winPLwQtzF56c5eQPws5dyN5LALeY/PgoDzUI0GA%2BcwiPnjzvx7rJFBOyTTlR8GgZJ3LvIZ4nBSu0sh3O7MWkALMi9eM7saJUCwxHwgImJnEwSYrQSW9zcyKgP6hARhQOMRHoNGqD0ieg9I%2BkU0GbAnB6RnwZsIyJOCfB6FMTRaFgCIBdMDI2C1aPoHoBvACF9wfaPtCyamBYg2A/AEYDEVjCPAbke7g0AMiNhB49ACAJoGWiaAbcro9RZovQQhBNA%2BULaHsCkWJAHoZrd8qJFmA6BgAI8XygpFmCkB7Ar0Cxfm1Zg3llowQggOEHzCjAqEcAlKAGCDCuj4YswZaB4k4W7B9AVycTN9A0AOLgAPoewDCM2ROBaADIaxHJAUjYiKBmI96DQNxE1NMwxUFQIwMSWNN/Acw4oFAGCj%2Bw/IVybFJhjIBVhgorCLAHUuWGNK7ggw4mPQGYDbhEkZS%2BcIk3A5K5Kl6scGEszaYUi5YbwXgSKN2ZMjRmxMVZgsumZ0iZB4yoUcLGWWCi5BMo3QfMulESiRBGgvZXMoZh3BnwmIoEfAPibLRVoXIcEPSOqBMBqgTQHxG5BkrNgHBPQE4DwnaiYAuoKgWSvoAIV3AiF4KfAHcEGifBeQ00IZg9y8C0KLw9Ik4A0BUCEiFo%2BgThdNEoUKKTgJwJoEwvxWEq/IPCxgHwpACbQYRcAWAEgFp4ek8mUi0YVYA5BXQGAjy55a8veWXBPl3yzMFkoxHlNsRtAgpUVBADFLLlxiBgREgxAyRrlWCu5YwDCARAbaXVDlS8reUVt9w93PlTwnhjSKwcA4TnCCr6hgqBEJC65ews4UTQHB9Ih7s2E%2BANA7VD3R1a2DJVrQNo0I7aFCvFUNAHBDQT4CcGbDNBUVdqpoM0HpExMVAIIxJuSq9UZMfVY0HoBNCmgzQU1zC%2BpowC8BYqSo%2BK25YgLjWgqYm2a/NaCLWhFqrkhMaGE0CAA%3D).

# Mocks & Expectations

In general, *mocks* are objects that simulate the behavior of a real implementation to the *system under test* (SUT),
while giving the *unit test* full control over their responses and behavior.

In *mimic++*, mocks are regular C++ objects.
They do not require a special framework context and can be constructed anywhere you need them.
Mocks are created using the `mimicpp::Mock` class template, which accepts one or more function types to define the mock's callable interface.
For example:
```cpp
mimicpp::Mock<void ()> myMock{};
```

This creates a mock object that exposes an `operator ()` with no parameters and `void` as return type (i.e. `void operator()()`).
The mock can be invoked like any other function object: `myMock();`

Calling mocks is only half the story, though.
By default, all calls to a mock are **rejected unless explicitly allowed**.
This is where *expectations* come in.

Expectations define **how and when** a mock may be called, and what should happen in response.
They are typically defined on a per-test-case basis.

{: .box-success}
Unlike some other mocking frameworks, *mimic++* enforces a strict expectation strategy.
Every mock call must match an existing expectation.
Otherwise, the call will be rejected, a violation will be reported, and the test will be aborted.

Expectations are also **scope-based**: they are verified during their destruction.
If an expectation is not fulfilled (i.e., not *satisfied*), it triggers a violation at the end of its lifetime.

We’ll explore how to define and manage expectations in the next section.

# A first test

Let’s now attempt to write our first unit test for the `Quiz` class.  
Before diving in, let’s briefly recap what `Quiz` does:
1. Accepts a non-empty collection of `QuizItem` objects.
2. Uses a provided `IOContext` to print questions and answer choices to the user.
3. Reads the user’s input from that same context.

This means we have
- a **fixed input** (the quiz items), and
- an **interface** (the `IOContext`), which — totally by coincidence 🙈 — is a perfect target for mocking.

## Test Strategy

We’ll write a test that
- sets up a quiz with one question,
- simulates a user answering it and
- verifies whether the `Quiz::run` result is correct.

Here’s the skeleton of our test:
```cpp
TEST_CASE("Quiz determines the correctness of the given answers.")
{
    mock_it_right::Quiz const quiz{
        {QuizItem{.question = "Q1", .choices = { "A1", "A2", "A3", "A4"}}}};

    auto const [expectedCorrectness, answer] = GENERATE(
        table<bool, char>({
            {false, 'B'},
            {false, 'c'},
            {false, 'd'},
            {true, 'A'},
            {true, 'a'}
        }));
   
    // (1) ...
    
    std::vector const result = quiz.run(ioCtx);

    REQUIRE(1u == result.size());
    CHECK(expectedCorrectness == result.front());
}
```

This single test case actually runs **five times**, using the Catch2 `GENERATE` macro.
Three of the inputs are **wrong answers** (`B`, `c`, `d`), and two are **correct** (`A`, `a`).

<div class="box-note">
   <b>NOTE</b> If you’re new to <i>Catch2</i>:
   <ul>
    <li><code>TEST_CASE</code> defines a test function.</li>
    <li><code>GENERATE</code> allows for parameterized test inputs — this test runs once for each row of the table.</li>
    <li><code>REQUIRE</code> is a fatal assertion — it stops the test immediately if it fails.</li>
    <li><code>CHECK</code> is a non-fatal assertion — it reports a failure but continues running the test.</li>
   </ul>
</div>

The missing part `// (1) ...` is where we’ll create a mock `ioCtx` and set up expectations to simulate the correct I/O behavior.

## The first Mock type

Let's take a look at how we can define an appropriate type for `ioCtx`, which provides a `read` method and two `write` method overloads.
Let's call it `IOContextMock`:
```cpp
struct IOContextMock
{
    mimicpp::Mock<char ()> read{};
    mimicpp::Mock<
        void (std::string_view), // the first overload
        void (std::span<std::string_view const, 4u>) // the second overload
        > write{};
};
```
As already stated, `mimicpp::Mock` supports overloading, which you can see in action in the `write` method case.

{: .box-note}
**NOTE** As mocks are just invocable objects, we exploit them here as *member functions*.
It's worth noting, that they are — from the language perspective — not actual functions;
they are still just (invocable) member objects.
Nevertheless, the user of such a class won't notice any difference when calling them,
except when they try to obtain a *member function pointer* to them, which is not possible.

<div class="box-note">
    <b>TIP</b> When an expectation violation occurs, the associated mock will appear in the test report.
    By default, mocks are named using the pattern <code>mimicpp::Mock&lt;Signatures+&gt;</code>,
    where <code>Signatures+</code> is replaced by the list of overloads.
    You can override this by explicitly assigning a name to the mock instance.
    For example: 
    <code>mimicpp::Mock&lt;char ()&gt; read&#123;&#123;.name = "IOContextMock::read"&#125;&#125;;</code>
</div>

## The first Expectation

Setting up expectations for a mock is a relatively straightforward process.
Let's begin by creating an expectation for the question text, which is provided to the `IOContext::write(std::string_view)` overload.
```cpp
IOContextMock ioCtx{}; // Create the mock object just as a standalone-mock by simply default constructing it.
mimicpp::ScopedExpectation questionTextExp = ioCtx.write.expect_call("Q1");
```

As you can see, creating an expectation is simply a matter of calling `.expect_call` on the mock object with the *correct* arguments.
Let’s now examine what *correct* actually means in this context.

### Introduction to Matchers

{: .box-note}
**NOTE** In the following, I'll refer to several sub-namespaces such as `matches`.
They’re actually defined inside the `mimicpp` namespace (e.g. `mimicpp::matches`),
but I recommend aliasing them into the global namespace.
This keeps your expectation-setup code concise and much more readable.

`expect_call` requires the same number of arguments as the corresponding `operator()`, but the argument types do not necessarily have to match exactly.
In fact, the arguments passed to `expect_call` are so-called *matchers*, which are responsible for verifying whether the given input is acceptable.

In general, *matchers* must be provided explicitly (e.g., `myMock.expect_call(mimicpp::eq(42))`, which expects the first argument to be equal to `42`).
However, *mimic++* supports a more concise syntax for the equality case, as used in our `write` expectation above.

So, the more verbose equivalent of our example would be:
`ioCtx.write.expect_call(matches::str::eq("Q1"))`, which simply states that the first argument must equal the string `"Q1"`.

<div class="box-note">
   <b>NOTE</b> This concise syntax will dispatch to the appropriate matcher, depending on the characteristics of the provided parameter:
   <ul>
    <li><code>matches::str::eq</code> is used when the parameter is a string type,</li>
    <li><code>matches::eq</code> is used when the parameter is equality-comparable,</li>
    <li>and it is rejected with a compile-error, when neither condition applies.</li>
   </ul>
</div>

### Managing Expectation-Lifetimes

Now let’s focus on the left-hand side of that expression:`mimicpp::ScopedExpectation questionTextExp`

`ScopedExpectation` is a closure type that stores the generated expectation and verifies it upon destruction.  
Since C++ (prior to C++26) doesn’t allow keeping objects with a
[placeholder-name](https://en.cppreference.com/w/cpp/language/conflicting_declarations.html#Potentially-conflict_declarations),
this often results in more verbose syntax than desired.
In general, it's not necessary to have access to an expectation object, thus always be required to name it uniquely is just noise.

To simplify this, *mimic++* provides the macro `SCOPED_EXP`(or its longer alias `MIMICPP_SCOPED_EXPECTATION`),
which handles creation and naming automatically.
Our example can therefore be simplified to: `SCOPED_EXP ioCtx.write.expect_call("Q1");`

## Building up more Expectations

Now let’s set up the `.write` call for the corresponding choices.
This time, we’ll provide an explicit matcher by hand: `matches::range::eq`, which requires component-wise equality.
```cpp
SCOPED_EXP ioCtx.write.expect_call(matches::range::eq(std::array{"A1", "A2", "A3", "A4"}));
```

Your container type needn’t match the parameter type exactly —
just ensure its elements are comparable for equality with the parameter container-type elements.

### Expectations and their Policies

Finally, let’s tackle the `.read` case by returning our test’s answer variable.
Previous expectations were trivial — they only required minimal setup and just returned void.
This time, our mock must **return a value** (`char`), so *mimic++* requires you to specify a return value when setting up the expectation.
Here’s where the so-called *expectation-policies* come into play:
they let you tweak an expectation’s behavior in various ways, giving you powerful control over multiple aspects.

{: .box-note}
**NOTE** *expectation-policies* is the umbrella term for features that modify expectation behavior.
Several policy categories exist, each affecting different aspects of an expectation.

Because our mock must return a `char`, we use the `finally::returns` (finalizer-)policy.
This finalizer takes a value and returns it whenever the expectation is matched.
```cpp
SCOPED_EXP ioCtx.read.expect_call()
    and finally::returns(answer);
```
Notice how you chain policies simply by using the overloaded `operator and` (which is just an alias for `operator &&`).

{: .box-note}
**NOTE** During the design process, I aimed to make the expectation-setup code as readable as possible,
without requiring too much knowledge about *mimic++* from the readers of the code.
Therefore, I introduced several policy sub-namespaces, which then almost form an English sentence for the construction.

### Sequencing Expectations

Are we done now? No!  
There is an important case missing: namely, the scenario where users enter an invalid answer that does not correspond to the given choices.
While this case is already handled by `Quiz::read_input`, it definitely needs to be tested.
But how can we do that?

In fact, we can achieve this by returning an incorrect character (e.g., `X`) and thus creating a separate expectation.
```cpp
SCOPED_EXP ioCtx.read.expect_call()
    and finally::returns('X');
```

This leads us to a crucial aspect of expectations:

{: .box-success}
By default, **all available expectations** are treated equally.
Therefore, if multiple matches are possible for any incoming call, *mimic++* will select any one of them.

In our case, since we only provide a single `QuizItem`,
we must ensure that the expectation for the invalid input is prioritized over the one for the valid input.
Otherwise, our `Quiz` will not prompt the user for another input,
leaving the expectations unfulfilled and resulting in a violation report.


This is where sequencing becomes essential, allowing users to create a reliable sequence of expectations.
Each expectation can only be matched when all of its predecessors are *satisfied*.
To facilitate this, *mimic++* provides a `mimicpp::SequenceT` type, which we can use to attach our expectations.
```cpp
mimicpp::SequenceT seq{};
SCOPED_EXP ioCtx.read.expect_call() // (1)
        and expect::in_sequence(seq)
        and finally::returns('X');
SCOPED_EXP ioCtx.read.expect_call() // (2)
        and expect::in_sequence(seq)
        and finally::returns(answer);
```

As you can see, we simply need to create a new sequence instance and attach the desired expectations using the `expect::in_sequence` policy.
This ensures that the first expectation is reliably matched before the second.

There is an even more convenient alternative that makes the sequence responsible for managing the lifecycle of the attached expectations.
This approach simplifies the process by abstracting some of the details.
```cpp
mimicpp::ScopedSequence seq{};
seq += ioCtx.read.expect_call() // (1)
    and finally::returns('X');
seq += ioCtx.read.expect_call() // (2)
    and finally::returns(answer);
```

This method is fully equivalent to the previous solution but offers a cleaner and more streamlined experience.

### Times Policies

There is still yet another case we should take into consideration:
internally, `Quiz::read_input` utilizes a loop, so it continues to ask for input as long as it receives invalid input.
We can, of course, add another expectation into our sequence, but that wouldn't scale well for more complex scenarios.

Therefore, *mimic++* provides the `expect::times` policy family,
which allows you to specify exactly how often an expectation should match until it is *satisfied*.
This can be done by simply adding that policy to our builder chain:
```cpp
seq += ioCtx.read.expect_call() // (1)
    and expect::times(3) // 3 times is just arbitrarily chosen here.
    and finally::returns('X');
```

Now, `Quiz::read_input` will receive the invalid input `X` exactly three times in a row until we supply the valid `answer`.

<div class="box-success">
   <code>expect::times</code> is a <i>times-policy</i> that can only be applied once for each expectation.
   If no such policy is specified, <i>mimic++</i> defaults to <code>expect::once</code>.
   Such a <i>times-policy</i> may also specify a range (e.g. <code>times::at_least(1)</code>),
    which denotes the minimum and maximum times an expectation must be matched.
   When an expectation matches the
   <ul>
    <li>minimum amount, it is called <i>satisfied</i>, and</li>
    <li>maximum amount, it is called <i>saturated</i>.</li>
   </ul>
</div>

When we reconsider our previous expectations,
it does not make sense for `Quiz` to ask for user input before the actual question and the choices have been printed,
so they are also good candidates for inclusion in the sequence.
To sum it up, the final setup that we can use to replace the `// (1) ...` marker in our test case looks like this:
```cpp
mimicpp::ScopedSequence ioSequence{};
IOContextMock ioCtx{};
ioSequence += ioCtx.write.expect_call("Q1");
ioSequence += ioCtx.write.expect_call(matches::range::eq(std::array{"A1", "A2", "A3", "A4"}));

// When we provide an invalid answer, we will get another chance.
// Let's force this three times in a row!
ioSequence += ioCtx.read.expect_call()
    and expect::times(3)
    and finally::returns('X');

// Let's now provide our actual answer.
ioSequence += ioCtx.read.expect_call()
    and finally::returns(answer);
```

# Conclusion

In this post, we explored the basics of how mocking can be done in *mimic++*.
This included creating mocks and setting up expectations.
Additionally, we touched on the topic of expectation policies and how sequences can be employed.

We tested our `Quiz` class, but we are not done yet!
As it stands, it is a very easy quiz since the correct answer is always `A`,
and the `QuizItems` always come in the same order.
Making it more interesting and creating the appropriate tests will be covered in a follow-up.
Until then, you might consider extending the Quiz App by adding these `QuizItems` into the item set.
But be careful!
The choices have been shuffled, so you need to determine the correct answer and place it in the 0th position.

| Question                                                         | A                          | B                               | C                                               | D                                     |
|------------------------------------------------------------------|----------------------------|---------------------------------|-------------------------------------------------|---------------------------------------|
| What is the default amount an expectation must match?            | Twice                      | Once                            | Never                                           | 42 times                              |
| Is there a specific order in which expectations must be matched? | Ordering?                  | In declaration order.           | In reverse declaration order.                   | No specific ordering required.        |
| What does it mean when an expectation is *satisfied*?            | Matched the minimum times. | Matched the maximum times.      | Matched at least once.                          | It's successfully linked with a mock. |
| What does it mean when an expectation is *saturated*?            | Matched the minimum times. | Matched the maximum times.      | Matched at least once.                          | It's successfully linked with a mock. |
| What is the role of the `SCOPED_EXP` macro?                      | Creates a new mock object. | Starts a new expectation setup. | Stores an expectation and manages its lifetime. | Invokes a mock object.                |
