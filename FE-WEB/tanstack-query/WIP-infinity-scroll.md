
### 무한 스크롤과 상태 복원

- 무한 스크롤은 새로고침하면 처음부터 보여진다. => query param에 url을 저장할 필요가 없다.
- 무한 스크롤이 존재하는 페이지에서 다른 페이지를 보다가 뒤로가기를 통해 다시 무한 스크롤이 존재하는 페이지로 돌아오면 이전 스크롤이 보여진다. => 마찬가지로 query param에 url을 저장할 필요가 없음. tanstack query가 알아서 복원함.

### QUESTION

- 뒤로가기를 통해서 재방문 했을 때 tanstack query가 알아서 복원할 수 있는 원리는 무엇인지?

[#2 React Query: Infinite Scroll - DEV Community](https://dev.to/kevin-uehara/2-react-query-infinite-scroll-1mg8)