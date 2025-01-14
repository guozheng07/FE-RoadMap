1. 封装为类
```
const { each, find } = require('lodash');
const swagger = require('swagger-client');

// 返回数据
interface InterfaceData {
  apis: any[]; // 各接口详细信息，数据需写入dev_yapi_doc表中
  cats: Record<string, any>[]; // 各接口所属的分组/模块详情，数据需写入dev_xx_cat_list表中
}

class SwaggerProcessor {
  // public author: string;  // 当前mis
  private swaggerData: any; // 通过swagger-client库解析后，还未加工的js对象数据
  // private isOAS3: boolean; // 是否是openapi3.0数据，是的话需要转化成swagger2.0数据
  
  constructor(swaggerData: any = null) {
    // this.author = author;
    this.swaggerData = swaggerData;
  }
  
  // 提供一个公共方法，获取从swagger2.0转化为js对象后，还未加工的数据
  public getSwaggerData(): any {
    return this.swaggerData;
  }

  /**
   * 路径标准化
   * @param path 初始路径
   * @returns 标准路径
   */
  private handlePath(path: string): string {
    // 不处理根路径
    if (path === '/') return path;
    // 路径以'/'开头
    if (path.charAt(0) !== '/') {
        path = '/' + path;
    }
    // 路径不以'/'结尾
    if (path.charAt(path.length - 1) === '/') {
        path = path.substr(0, path.length - 1);
    }
    return path;
  }

  /**
   * 获取结构化数据
   * @param definitions 结构化对象
   * @param ref schema ref
   * @returns 请求参数、响应参数在dev_xx_cat_list表中的值结构
   */
  private handleSchemaData(definitions: any, ref: string): any {
    const result: any = {};
    const refData = definitions[ref];
    if (!refData) return result;
    const dataObj: any = Object.keys(refData?.properties || {}).map((item) => {
      const baseObj: any = {
        name: item,
        dataTypeFe: refData?.properties?.[item]?.type,
        isRequired: (refData?.required || []).includes(item),
        description: refData?.properties?.[item]?.description || refData?.properties?.[item]?.title,
        children: [],
      };
      // 字段有结构化数据
      if (refData?.properties?.[item]?.$$ref) {
        baseObj.children = this.handleSchemaData(definitions, refData.properties[item].$$ref?.split('/').pop());
      }
      // 字段为数组 && 存在items
      // 示例：
      // categoryIdList:
      //   type: array
      //   description: 二级分类list
      //   items:
      //     type: integer
      //     format: int64
      if (refData?.properties?.[item]?.type === 'array' && refData?.properties?.[item]?.items) {
        if (refData.properties[item].items?.$$ref) {
          baseObj.children = baseObj.children.concat(
            this.handleSchemaData(definitions, refData.properties[item].items.$$ref?.split('/').pop())
          );
        } else {
          baseObj.children = baseObj.children.concat({
            name: 'items',
            dataTypeFe: refData.properties[item].items?.type,
            isRequired: false,
            description: '',
            children: [],
          });
        }
      }

      return baseObj;
    });

    return dataObj;
  }

  /**
   * 处理请求参数
   * @param apiInfo 接口信息
   * @param definitions 结构化对象
   * @returns 请求参数
   */
  private handleRequest(apiInfo: any, definitions: any): string {
    const apiList = apiInfo.parameters || [];
    // console.log(apiList, 'apiList');

    let query_parameters: any = [];
    apiList.forEach((item) => {
      const queryObj = {
        name: item.name,
        dataTypeFe: item.type,
        isRequired: item.required,
        description: item.description,
        children: [],
      };
      if (item.schema && item.schema.$$ref) { // 请求参数是结构化对象
        queryObj.children = this.handleSchemaData(definitions, item.schema.$$ref.split('/').pop());
        queryObj.dataTypeFe = 'object';
      }
      query_parameters.push(queryObj);
    });

    return JSON.stringify(query_parameters);
    // return query_parameters;
  }

  /**
   * 处理响应参数
   * @param apiInfo 接口信息
   * responses:
        '200':
          description: OK
          schema:
            $$ref: '#/definitions/统一返回对象'
        '201':
          description: Created
        '401':
          description: Unauthorized
        '403':
          description: Forbidden
        '404':
          description: Not Found
   * @param definitions 结构化对象
   * @returns 响应参数
   */
  private handleResponse(apiInfo: any, definitions: any): string {
    const api = apiInfo.responses || {};
    
    let response_params: any = [];
    if (api && typeof api === 'object') {
      let codes = Object.keys(api); // 各响应码
      let curCode; // 请求成功（有效）的响应码
      if (codes.length > 0) {
        if (codes.indexOf('200') > -1) {
          curCode = '200';
        } else curCode = codes[0]; // 响应码没有'200'，则数组第一项默认为有效的响应码
        
        let res = api[curCode];
        if (res && typeof res === 'object') {
          if (res.schema && res.schema.$$ref) { // 响应参数是结构化对象
            response_params = this.handleSchemaData(definitions, res.schema.$$ref.split('/').pop());
          }
        }
      }
    }
    return JSON.stringify(response_params);
    // return response_params;
  }

  /**
   * 读取和解析 swagger 规范的文档，将其转化为 js 对象
   * @param res swagger 文档
   * @returns object
   */
  private async handleSwaggerData(res: any): Promise<any> {
    return await new Promise((resolve) => {
      let data = swagger({
          spec: res,
      });

      data.then((res) => {
        resolve(res.spec);
      });
    });
  }

  /**
   * 加工/处理 js 对象，使字段结构更符合数据库中表的字段结构
   * @param data 接口信息
   * @returns 写入dev_yapi_doc表的信息
   */
  private handleSwagger(data: any, definitions: any): any {
      let api: any = {};
      // 接口基本信息
      api.path = this.handlePath(data.path);
      api.method = data.method.toUpperCase();
      api.description = data.summary;
      api.display_name = data.summary;
      api.name = data.operationId; // 唯一值，可用于更新
      api.cat_name = data.tags && Array.isArray(data.tags) ? data.tags[0] : null;
      api.deprecated = data.deprecated || false; // 接口是否弃用，若为true，则弃用

      // 处理请求参数
      api.queryParameters = this.handleRequest(data, definitions);
      // 处理响应参数
      api.responseParams = this.handleResponse(data, definitions);
      // todo：后续生成mock数据
      // 请求mock数据
      api.requestMock = JSON.stringify({});
      // 响应mock数据
      api.responseMock = JSON.stringify({});

      return api;
    }

  /**
   * 主运行函数
   * @param res 接口原始数据（例如https://1975-gxkra-sl-manager.mall.test.sankuai.com/api/m/pms/v2/api-docs返回的数据）
   * @returns 写入dev_yapi_doc、dev_xx_cat_list表的信息
   */
  public async process(res: any): Promise<any> {
    let interfaceData: InterfaceData = { apis: [], cats: [] };
    if (typeof res === 'string' && res) {
      try {
        res = JSON.parse(res);
      } catch (e: any) {
        console.error('json 解析出错', e.message);
      }
    }

    // todo：确认是否需要handleSwaggerData函数来处理，感觉上面JSON.parse后，数据就可用了（从json->js对象）
    res = await this.handleSwaggerData(res.data);
    // console.log(res, '得到swagger-client解析后的数据，和在https://editor-next.swagger.io/中解析得到数据一致');
    this.swaggerData = res;

    // 1.顶层有tags字段时，先填充cats数组
    if (Array.isArray(res?.tags)) {
      res.tags.forEach((tag) => {
        interfaceData.cats.push({
          cat_name: tag.name,
          service_name: tag.description,
        });
      });
    }

    /**
     * apis：{ '/auditConfig/create': {} }
     * path：'/auditConfig/create'
     */
    each(res.paths, (apis, path) => {
      /**
       * api：{ 'post': {} } // 请求方法的键值对，才是具体的接口信息
       * method：'post'
       */
      each(apis, (api, method) => {
        api.path = path;
        api.method = method;
        let data: any = null;
        try {
          data = this.handleSwagger(api, res.definitions);
          if (data.cat_name) {
            // 2.顶层的paths字段中，各接口的tags字段里有顶层的tags字段中不存在的接口分组信息，则补充填充cats数组
            if (!find(interfaceData.cats, (item) => item.cat_name === data.cat_name)) {
              interfaceData.cats.push({
                cat_name: data.cat_name,
                service_name: data.cat_name,
              });
            }
          }
        } catch (err) {
          data = null;
        }
        // 接口信息存在 && 接口有效
        if (data && data.deprecated === false) {
          // 3.填充apis数组
          interfaceData.apis.push(data);
        }
      });
    });

    return interfaceData;
  }
}

export default SwaggerProcessor;
```
2. 接口调用
```
import SwaggerProcessor from './swagger';

async xxParseSwaggerData(params) {
    const author = params?.userInfo?.login;
    const { url, projectId, teamId } = params?.request?.body || {};
    if (!url) {
      throw new Error(`需要传入 url`);
    }
    // 这里需要去判断域名是否合法
    if (!url.includes('sankuai.com') && !url.includes('sankuai.net') && !url.includes('meituan.com') && !url.includes('meituan.net')) {
      throw new Error(`域名不合法，请重新输入链接`)
    }

    const swaggerProcessor = new SwaggerProcessor();
    try {
      const originalData = await axios.get(url);
      const { apis, cats } = await swaggerProcessor.process(originalData);

      if (!cats.length || !apis.length) {
        return {
          msg: '解析 swagger 数据出错，请重试！',
        };
      }

      const formatDate = (date) => {
        const pad = (num) => (num < 10 ? '0' : '') + num;
        return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}:${pad(date.getSeconds())}`;
      };
      const currentTime = formatDate(new Date());

      /**
       * 模式：
       * - 完全覆盖：先删除，再插入
       */

      // 删除当前项目下的 dev_xx_cat_list 表数据
      const deleteCatSql = `
        DELETE FROM dev_xx_cat_list WHERE project_id = '${projectId}';
      `;
      await this.app.sequelizeDB.query(deleteCatSql, {
        raw: true,
        type: QueryTypes.DELETE,
      });

      // cats 插入 dev_xx_cat_list 表
      const catInsertSql = `
        INSERT INTO dev_xx_cat_list (project_id, cat_name, service_name, add_time, update_time)
        VALUES ${cats.map((cat) => `('${projectId}', '${cat.cat_name}', '${cat.service_name}', '${currentTime}', '${currentTime}')`).join(', ')};
      `;
      await this.app.sequelizeDB.query(catInsertSql, {
        raw: true,
        type: QueryTypes.INSERT,
      });

      // 获取本次插入的 cat_name 对应的 cat_id
      const catIds = await this.app.sequelizeDB.query(`
        SELECT id, cat_name FROM dev_xx_cat_list WHERE cat_name IN (${cats.map((cat) => `'${cat.cat_name}'`).join(', ')});
      `, {
        raw: true,
        type: QueryTypes.SELECT,
      });
      const catIdMap = {};
      catIds.forEach((cat) => {
        catIdMap[cat.cat_name] = cat.id;
      });

      // 删除当前项目下的 dev_yapi_doc表数据
      const deleteApiSql = `
        DELETE FROM dev_yapi_doc WHERE projectId = '${projectId}';
      `;
      await this.app.sequelizeDB.query(deleteApiSql, {
        raw: true,
        type: QueryTypes.DELETE,
      });

      // apis 插入 dev_yapi_doc 表
      const apiInsertSql = `
        INSERT INTO dev_yapi_doc (projectId, name, path, method, queryParameters, responseParams, requestMock, responseMock,
          createdAt, updatedAt, team_id, description, display_name, author, api_source, serviceName, cat_id)
        VALUES ${apis.map((api) => `('${projectId}', '${api.name}', '${api.path}', '${api.method}', '${api.queryParameters}', 
          '${api.responseParams}', '${api.requestMock}', '${api.responseMock}', '${currentTime}', '${currentTime}',
          '${teamId}', '${api.description}', '${api.display_name}', '${author}', ${API_SOURCE.SWAGGER}, '${api.cat_name}', '${catIdMap[api.cat_name]}')`).join(', ')};
      `;
      await this.app.sequelizeDB.query(apiInsertSql, {
        raw: true,
        type: QueryTypes.INSERT,
      });

      return {
        data: '解析完成',
      };
    } catch (error: any) {
      console.error('swagger 数据写入出错：', error);
      return {
        msg: 'swagger 数据写入出错，请重试！',
      };
    }
  }
```
