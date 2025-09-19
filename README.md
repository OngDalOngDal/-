<글쓰기/글보기>

memberMapper.xml 에서 lat, lon 추가

<select id="findAddressByMemberId" resultType="com.disaster.domain.MemberAddressDTO">
    	SELECT 
        	address_id as addressId, 
        	member_id as memberId, 
        	muni_code as muniCode,
        	lat,
        	lon
    	FROM 
        	member_address
    	WHERE 
        	member_id = #{memberId}
    	LIMIT 1 <!-- 한 멤버가 여러 주소를 가질 경우를 대비해 대표 주소 하나만 가져옴 -->
	</select>

------------------------------------------------------------------------

WriteDTO.java

private LocalDateTime createdAt;  으로 변경
