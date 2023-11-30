    @Test
    public void parse() {
        Map<String,RouteParameter> params;
        RouteParameter param;
        
        // no named parameters is null
        params = RouteParameter.parse("/user");
        assertThat(params, aMapWithSize(0));
        
        params = RouteParameter.parse("/user/{id}/{email: [0-9]+}");
        
        param = params.get("id");
        assertThat(param.getName(), is("id"));
        assertThat(param.getToken(), is("{id}"));
        assertThat(param.getRegex(), is(nullValue()));
        
        param = params.get("email");
        assertThat(param.getName(), is("email"));
        assertThat(param.getToken(), is("{email: [0-9]+}"));
        assertThat(param.getRegex(), is("[0-9]+"));
    }