package brickhouse.hbase;
/**
 * Copyright 2012 Klout, Inc
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 **/

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.HTable;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hive.ql.exec.Description;
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.log4j.Logger;

import java.io.IOException;
import java.util.*;
import java.util.Map.Entry;

/**
 *  Simple UDF for doing single PUT into HBase table.
 *  NOTE: Not intended for doing massive reads from HBase, but only when relatively few rows are being read.
 *
 */
@Description(name="hbase_put",
    value = "string _FUNC_(config, map<string, string> key_value) - \n" +
        "string _FUNC_(config, key, value) - Do a HBase Put on a table. " +
        " Config must contain zookeeper \n" +
        "quorum, table name, column, and qualifier. Example of usage: \n" +
        "  hbase_put(map('hbase.zookeeper.quorum', 'hb-zoo1,hb-zoo2', \n" +
        "                'table_name', 'metrics', \n" +
        "                'family', 'c', \n" +
        "                'qualifier', 'q'), \n" +
        "            'test.prod.visits.total', \n" +
        "            '123456') "
)
public class PutUDF extends UDF {
  private static final Logger LOG = Logger.getLogger(PutUDF.class);

  static private String FAMILY_TAG = "family";
  static private String QUALIFIER_TAG = "qualifier";
  static private String TABLE_NAME_TAG = "table_name";
  static private String ZOOKEEPER_QUORUM_TAG = "hbase.zookeeper.quorum";

  private static Map<String, HTable> htableMap = new HashMap<String,HTable>();
  private static Configuration config = new Configuration(true);

  public String evaluate(Map<String, String> configIn, String key, String value) {
    checkConfig(configIn);

    try {
      HTable table = getHTable(configIn.get(TABLE_NAME_TAG), configIn.get(ZOOKEEPER_QUORUM_TAG));
      Put thePut = new Put(key.getBytes());
      thePut.add(configIn.get(FAMILY_TAG).getBytes(), configIn.get(QUALIFIER_TAG).getBytes(), value.getBytes());

      table.put(thePut);
      return "Put " + key + ":" + value;
    } catch(Exception exc) {
      LOG.error("Error while doing HBase Puts");
      throw new RuntimeException(exc);
    }
  }

  public String evaluate(Map<String, String> configIn, Map<String, String> keyValueMap) {
    checkConfig(configIn);

    try {
      List<Put> putList = new ArrayList<Put>();
      for (Map.Entry<String, String> keyValue : keyValueMap.entrySet()) {
        Put thePut = new Put(keyValue.getKey().getBytes());
        thePut.add(configIn.get(FAMILY_TAG).getBytes(),
                   configIn.get(QUALIFIER_TAG).getBytes(),
                   keyValue.getValue().getBytes());
        putList.add(thePut);
      }

      HTable table = getHTable(configIn.get(TABLE_NAME_TAG), configIn.get(ZOOKEEPER_QUORUM_TAG));
      table.put(putList);
      return "Put " + keyValueMap.toString();
    } catch(Exception exc) {
      LOG.error("Error while doing HBase Puts");
      throw new RuntimeException(exc);
    }
  }

  private HTable getHTable(String tableName, String zkQuorum) throws IOException {
    HTable table = htableMap.get(tableName);
    if(table == null) {
      config = new Configuration(true);
      Iterator<Entry<String,String>> iter = config.iterator();
      while(iter.hasNext()) {
        Entry<String,String> entry =  iter.next();
        LOG.info("BEFORE CONFIG = " + entry.getKey() + " == " + entry.getValue());
      }
      config.set("hbase.zookeeper.quorum", zkQuorum);
      Configuration hbConfig = HBaseConfiguration.create(config);
      iter = hbConfig.iterator();
      while(iter.hasNext()) {
        Entry<String,String> entry =  iter.next();
        LOG.info("AFTER CONFIG = " + entry.getKey() + " == " + entry.getValue());
      }
      table = new HTable(hbConfig, tableName);
      htableMap.put(tableName, table);
    }

    return table;
  }

  /**
   * Throws RuntimeException if config is incomplete.
   * @param configIn
   */
  private void checkConfig(Map<String, String> configIn) {
    if (!configIn.containsKey(FAMILY_TAG) ||
        !configIn.containsKey(QUALIFIER_TAG) ||
        !configIn.containsKey(TABLE_NAME_TAG) ||
        !configIn.containsKey(ZOOKEEPER_QUORUM_TAG)) {
      String errorMsg = "Error while doing HBase Puts. Config is missing for: " + FAMILY_TAG + " or " +
          QUALIFIER_TAG + " or " + TABLE_NAME_TAG + " or " + ZOOKEEPER_QUORUM_TAG;
      LOG.error(errorMsg);
      throw new RuntimeException(errorMsg);
    }
  }
}
