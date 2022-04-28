<script lang="ts">
  import type { Users, User, BlockUsers, BlockUser } from "src/types"

  import { fade, slide } from "svelte/transition"
  const duration = { duration: 300 }

  import ProgressCircle from "svelte-progresscircle"
  import type { FullUser } from "twitter-d"

  //NOTE テスト用
  // import testdata from "../../testdata.json"
  const testdata = { following: {}, search: {} }
  const localDebug = false

  export let cookie

  function post(url) {
    const u = new URL("https://blocker.hmpf.club" + url)
    u.searchParams.set("access_token", cookie["access_token"])
    u.searchParams.set("access_token_secret", cookie["access_token_secret"])
    url = u.toString()
    return fetch(url, {
      method: "POST",
    })
  }
  function get(url) {
    const u = new URL("https://blocker.hmpf.club" + url)
    u.searchParams.set("access_token", cookie["access_token"])
    u.searchParams.set("access_token_secret", cookie["access_token_secret"])
    url = u.toString()
    return fetch(url, {
      method: "GET",
    })
  }

  let blocklist =
    "1517951340336402432,1515252940608778240,1235518101501796352,1518263222188802048,1515660158118273028,1516422989642092549,1516426031405875205,1329051155012276225,1519507103261462529"
  let blocktype: "screen_name" | "id" = "id"
  let blocking = false
  let blockProgress = 0
  let promiseBlockUsers: Promise<BlockUsers> = null
  let blockFailureList: {
    accountId: string
    code: number
    message: string
  }[] = []
  let openBlockFailureInfo = false

  $: blockIds = blocklist.split(",").filter((v) => v !== "")

  async function block() {
    console.log(blocktype)
    blocking = true
    blockFailureList = []
    openBlockFailureInfo = false
    promiseBlockUsers = new Promise<BlockUsers>(async (resolve) => {
      blockProgress = 0
      const result: BlockUsers = {}
      await Promise.all(
        blockIds.map(async (id) => {
          const data = localDebug
            ? new Response(
                await new Promise((resolve) =>
                  setTimeout(
                    () => resolve(JSON.stringify(testdata.following)),
                    1000
                  )
                )
              )
            : await post("/block?" + blocktype + "=" + id)
          const { statusText } = data
          const dataJson:
            | FullUser
            | {
                errors: {
                  code: number
                  message: string
                }[]
              } = await data.json()
          const ok = !("errors" in dataJson)
          result[id] = {
            data: ok ? dataJson : undefined,
            ok,
            statusText,
          }
          if (!ok) {
            const { errors } = dataJson
            blockFailureList = [
              ...blockFailureList,
              {
                accountId: id,
                code: errors[0].code,
                message: errors[0].message,
              },
            ]
          }
          blockProgress++
        })
      )
      setTimeout(() => {
        blocking = false
        promiseBlockUsers = null
      }, 2400)
      resolve(result)
    })
  }
</script>

<header>
  <h1>CoCntbrBlocker</h1>
  <!-- <a href="/?logout=true">ログアウト</a> -->
</header>

<main id="logged">
  {#if promiseBlockUsers !== null}
    <div
      class="block_progress"
      transition:slide={{ duration: duration.duration * 2 }}
    >
      <ProgressCircle max={blockIds.length} value={blockProgress}>
        {#await promiseBlockUsers}
          <span class="block_progress_text" in:fade={duration}
            ><span>{Math.round((blockProgress / blockIds.length) * 100)}</span
            >%</span
          >
        {:then}
          <svg
            class="block_complete_icon"
            transition:slide={duration}
            xmlns="http://www.w3.org/2000/svg"
            viewBox="0 0 24 24"
            ><path d="M0 0h24v24H0V0z" fill="none" /><path
              d="M9 16.2L4.8 12l-1.4 1.4L9 19 21 7l-1.4-1.4L9 16.2z"
            /></svg
          >
        {/await}
      </ProgressCircle>
    </div>
    {#await promiseBlockUsers}
      <div class="users_count" in:slide={duration}>
        <span>{blockIds.length}</span>人をブロックしています……
      </div>
    {:then blockUsers}
      <div class="users_count" in:fade={duration} out:slide={duration}>
        <span>{Object.values(blockUsers).filter(({ ok }) => ok).length}</span
        >人をブロックしました
      </div>
    {/await}
  {/if}
  {#if blockFailureList.length}
    <div class="block_failure_container" transition:slide={duration}>
      <button
        class="block_failure_btn"
        on:click={() => (openBlockFailureInfo = !openBlockFailureInfo)}
        ><span>{blockFailureList.length}人のブロックに失敗しました</span><i
          >{openBlockFailureInfo ? "expand_less" : "expand_more"}</i
        ></button
      >
      {#if openBlockFailureInfo}
        <div class="block_failure_info" transition:slide={duration}>
          <ul>
            {#each blockFailureList as blockFailure}
              <li>
                <dl>
                  <dt>{blockFailure.accountId}</dt>
                  <dd>{blockFailure.message}</dd>
                </dl>
              </li>
            {/each}
          </ul>
        </div>
      {/if}
    </div>
  {/if}
  {#if promiseBlockUsers === null}
    <div
      class="container"
      transition:slide={{ duration: duration.duration * 2 }}
    >
      <div class="coc_info">
        <div class="info_text">
          <p>CoCシナリオネタバレアカウントを一括ブロックすることが出来ます。</p>
          <p>
            ※アカウント削除済のアカウントはブロックできない為、登録数と実際のブロック数が異なる場合があります。<br
            />
            最新情報は開発アカウントをご確認ください。
          </p>
        </div>
        <div class="links">
          <ul>
            <li>
              <a
                href="https://twitter.com/CoCntbrBlocker"
                target="_blank"
                rel="noopener">開発アカウント</a
              >
            </li>
            <li>
              <a
                href="https://l1n4r1a.booth.pm/items/3813812"
                target="_blank"
                rel="noopener">作者を支援する(投げ銭)</a
              >
            </li>
          </ul>
          <ul>
            <li>
              <a href="https://lunachi.me" target="_blank" rel="noopener"
                >るなちのサイト</a
              >
            </li>
            <li>
              <a
                href="https://lunachi.me/cocntbrblocker/"
                target="_blank"
                rel="noopener">このツールの使い方</a
              >
            </li>
          </ul>
        </div>
      </div>
      <div class="radio_container">
        <div
          class="sn_id_radio"
          on:click={() => {
            blocklist = ""
          }}
        >
          <label class={blocktype === "screen_name" ? "checked" : ""}>
            <input
              type="radio"
              id="screen_name"
              name="type"
              value="screen_name"
              checked
              bind:group={blocktype}
            />Screen Name (@ユーザー名)
          </label>
          <label class={blocktype === "id" ? "checked" : ""}>
            <input
              type="radio"
              id="id"
              name="type"
              value="id"
              bind:group={blocktype}
            />
            User ID (ユーザー固有のID)
          </label>
        </div>
        <textarea class="block_users_textarea" bind:value={blocklist} />
      </div>
    </div>
  {/if}

  <div class="sticky_container">
    <div class="top_cover">
      <button
        class="block"
        on:click={() => {
          const scroll = document.body.animate(
            [{ marginTop: `-${window.pageYOffset - 1}px` }, { marginTop: 0 }],
            {
              duration: duration.duration,
              easing: "ease-in-out",
            }
          )
          window.scroll(0, 0)
          blocking = true
          scroll.addEventListener("finish", () => {
            block()
          })
        }}
        disabled={!Boolean(blocklist) || blocking}
        >{blockIds.length
          ? `${blockIds.length}人をブロックする`
          : `${blocktype}が指定されていません`}</button
      >
    </div>
  </div>
</main>
<footer>
  <p>ソースコード</p>
  <p>
    <a
      href="https://github.com/Steve-0628/blocklist-svelte"
      target="_blank"
      rel="noopener">フロントエンド</a
    >
    <a
      href="https://github.com/Steve-0628/blocker"
      target="_blank"
      rel="noopener">バックエンド</a
    >
  </p>
</footer>
<!-- <footer>
  <iframe loading="lazy" src={footerUrl} title="footer" />
</footer>
<iframe src={sideUrl} title="side" /> -->
