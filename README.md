<댓글 대댓글 삭제관련>

communityDetail.html

<div class="comment-actions">
    <button type="button" class="reply-btn">답글</button>
    <button type="button" class="reply-btn">신고</button>
에서
<div class="comment-actions">
    <button type="button" class="reply-btn">답글</button>
    <button type="button" class="reply-btn"
            th:if="${#authorization.expression('isAuthenticated()') and #authentication.principal.memberId != cmt.memberId}">신고</button>
변경


<div class="comment-actions">
    <button type="button" class="reply-btn">신고</button>
에서
<div class="comment-actions">
    <button type="button" class="reply-btn"
            th:if="${#authorization.expression('isAuthenticated()') and #authentication.principal.memberId != reply.memberId}">신고</button>
변경


------------------------------------------------------------------------

CommentMapper.xml - 댓글 삭제 시 삭제된 댓글이라고 안나옴

<select id="findByPostId" resultMap="commentMap">
        SELECT c.comment_id,
               c.post_id,
               c.member_id,
               c.parent_id,
               c.body,
               c.status,
               c.created_at,
               m.nickname
        FROM comment c
        JOIN member m ON c.member_id = m.member_id
        WHERE c.post_id = #{postId}
          AND (c.status IS NULL OR c.status != 'DELETED')
        ORDER BY IF(c.parent_id IS NULL, c.comment_id, c.parent_id), c.created_at
    </select>

에서

<select id="findByPostId" resultMap="commentMap">
    	SELECT c.comment_id,
           c.post_id,
           c.member_id,
           c.parent_id,
           c.body,
           c.status,
           c.created_at,
           m.nickname
    	FROM comment c
    	JOIN member m ON c.member_id = m.member_id
    	WHERE c.post_id = #{postId}
    	ORDER BY IF(c.parent_id IS NULL, c.comment_id, c.parent_id), c.created_at
	</select>
변경

------------------------------------------------------------------------

CommentMapper.java 추가함
/** 댓글을 'DELETED' 상태로 업데이트 */
    void updateCommentStatusToDelete(@Param("commentId") Long commentId);
/** 댓글 ID로 작성자의 member_id를 조회 (권한 확인용) */
    Long findMemberIdByCommentId(@Param("commentId") Long commentId);

------------------------------------------------------------------------

CommentMapper.xml 추가

<!-- ✅ 댓글의 status를 'DELETED'로 변경하는 쿼리 -->
    <update id="updateCommentStatusToDelete">
        UPDATE comment
        SET status = 'DELETED'
        WHERE comment_id = #{commentId}
    </update>

<!-- ✅ 댓글 ID로 작성자의 member_id를 찾는 쿼리 -->
    <select id="findMemberIdByCommentId" resultType="long">
        SELECT member_id FROM comment WHERE comment_id = #{commentId}
    </select>

------------------------------------------------------------------------

CommentService.java

/** 댓글 삭제 (소프트 삭제) */
    public void deleteComment(Long commentId) {
        // ✅ Mapper의 updateStatus 호출
        commentMapper.updateStatus(commentId, "DELETED");
    }

에서

/** 댓글 삭제 (소프트 삭제) */
    public void deleteComment(Long commentId, Long currentMemberId) {
        Long authorMemberId = commentMapper.findMemberIdByCommentId(commentId);
        
        // 권한 확인: 현재 로그인한 사용자가 댓글 작성자인지 확인
        if (authorMemberId == null || !authorMemberId.equals(currentMemberId)) {
            // 권한이 없으면 예외를 발생시켜 작업을 중단
            throw new AccessDeniedException("댓글을 삭제할 권한이 없습니다.");
        }
        commentMapper.updateStatus(commentId, "DELETED");
    }

변경

------------------------------------------------------------------------

CommentController.java

/** ✅ 댓글 삭제 */
    @PostMapping("/delete/{commentId}")
    public String deleteComment(@PathVariable("commentId") Long commentId,
                                @RequestParam("postId") Long postId) {
        // TODO: 본인 댓글만 삭제할 수 있도록 권한 체크 로직 추가 필요
        commentService.deleteComment(commentId);
        return "redirect:/community/communityDetail/" + postId;
    }

에서

@PostMapping("/delete/{commentId}")
    public String deleteComment(@PathVariable("commentId") Long commentId, 
                                @RequestParam("postId") Long postId,
                                Authentication authentication,
                                RedirectAttributes redirectAttributes) {

        try {
            CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();
            Long currentMemberId = userDetails.getMemberId();
            
            commentService.deleteComment(commentId, currentMemberId);
            redirectAttributes.addFlashAttribute("message", "댓글이 삭제되었습니다.");

        } catch (AccessDeniedException e) {
            redirectAttributes.addFlashAttribute("error", "댓글을 삭제할 권한이 없습니다.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", "댓글 삭제 중 오류가 발생했습니다.");
        }
        
        return "redirect:/community/communityDetail/" + postId;
    }

변경

------------------------------------------------------------------------

communityDetail.html

<div class="comment-list"> 쪽 아래처럼 변경

<div class="reply-list">
            <div th:each="reply : ${comments}" th:if="${reply.parentId == cmt.commentId}" class="comment reply">
                <!-- 삭제된 대댓글 처리 -->
                <div th:if="${reply.status == 'DELETED'}">
                     <p class="comment-body" style="color:#999;">(삭제된 댓글입니다)</p>
                </div>
                <!-- 정상 대댓글 처리 -->
                <div th:unless="${reply.status == 'DELETED'}">
                    <div class="comment-meta">
                        <span class="comment-author" th:text="${reply.nickname}"></span>
                        <span class="comment-date" th:text="${#temporals.format(reply.createdAt,'yyyy.MM.dd HH:mm')}"></span>
                    </div>
                    <p class="comment-body" th:text="${reply.body}"></p>
                    <div class="comment-actions">
                        <button type="button" class="reply-btn">신고</button>
                        <!-- 대댓글 삭제 버튼 -->
                        <form th:if="${#authorization.expression('isAuthenticated()') and #authentication.principal.memberId == reply.memberId}"
                              th:action="@{/comment/delete/{commentId}(commentId=${reply.commentId})}" method="post" style="display:inline;">
                            <input type="hidden" name="postId" th:value="${post.postId}">
                            <button type="submit" class="reply-btn" onclick="return confirm('정말로 삭제하시겠습니까?');">삭제</button>
                        </form>
                    </div>
                </div>
            </div>
        </div>
