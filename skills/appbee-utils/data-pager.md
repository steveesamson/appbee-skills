# utils.useDataPager

Walks every record that matches a filter, one page at a time. It's for workers and startup routines that must re-enqueue or reprocess a whole table without loading it at once.

```ts
const req = { source: utils.useSource('core') };
const queue = workerState().useQueue(MAIL_CREATED);

const pager = utils.useDataPager<Outmail>({
  model: workerState().model.Outmails(req),
  params: { sent: false },              // find query; operator keys work: { 'users<': 1 }
  includes: 'id, to, subject',          // optional projection; default all fields
  LIMIT: 200,                           // page size, default 200
  debug: false,
  onPage: (rows, next) => {
    for (const row of rows) queue.addJob({ ...row, id: `${row.id}` });
    next?.();
  },
});
await pager.start();
```

- **`onPage(rows, next)`** receives each page. `next` is `undefined` on the last page. Call `next?.()` to continue; skipping the call stops paging, which is also how to stop early.
- **`start()`** resolves after the **first** page is handed to `onPage`. Later pages run in the background as `next()` is called, so don't treat its resolution as "all pages done".
- **Errors.** A `find` error calls `onPage([])` once, with no `next`.
- **Mutating the filter.** When the work changes the filtered field (for example, setting `sent: true` on processed rows), offset paging skips rows, because the matching set shrinks under the cursor. Page a stable filter, or update after paging completes.
